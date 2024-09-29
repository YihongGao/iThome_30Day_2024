
# Day-24 服務的部署策略 - Ingress NGINX Controller

# 前言
昨天初步介紹了各種服務部署策略，其中有兩項是 Kubernetes Deployment 原生不支援的
- 藍綠部署（Blue/Green）
- 金絲雀部署（Canary）

今天我們搭配 [Ingress NGINX Controller](https://github.com/kubernetes/ingress-nginx/tree/main) 來實現這兩個部署策略

# 環境準備
1. 安裝 Ingress NGINX Controller 到本地 kind 環境  

參考 [2023年/Day11文章](Day-11-Kubernetes_介紹-Ingress) 或 執行以下指令
```shell
cat <<EOF > kind-config.yaml
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
nodes:
- role: control-plane
  kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 30000
    hostPort: 30000
    listenAddress: "0.0.0.0" # Optional, defaults to "0.0.0.0"
    protocol: tcp # Optional, defaults to tcp
  - containerPort: 30001
    hostPort: 30001
    listenAddress: "0.0.0.0" # Optional, defaults to "0.0.0.0"
    protocol: tcp # Optional, defaults to tcp
EOF

kind create cluster --config kind-config.yaml

kubectl apply --filename https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/kind/deploy.yaml

kubectl patch -n ingress-nginx service ingress-nginx-controller --type='json' -p='[
  {"op": "replace", "path": "/spec/ports/0/nodePort", "value": 30000},
  {"op": "replace", "path": "/spec/ports/1/nodePort", "value": 30001}
]'
```

# 部署 demo 服務
```shell
kubectl apply https://raw.githubusercontent.com/YihongGao/iThome_30Day_2024/refs/heads/main/resources/day24/apps/overlays/production/deploy.yml
```
能看到 `ithome` namespace 中有兩個 Deployment, Service
```shell
kubectl get deployments.apps,service,ep 

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/app-backend          2/2     2            2           36m
deployment.apps/app-backend-canary   2/2     2            2           36m

NAME                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/app-backend          ClusterIP   10.96.108.55    <none>        80/TCP    47m
service/app-backend-canary   ClusterIP   10.96.227.207   <none>        80/TCP    47m

NAME                           ENDPOINTS                           AGE
endpoints/app-backend          10.244.0.34:8080,10.244.0.35:8080   47m
endpoints/app-backend-canary   10.244.0.29:8080,10.244.0.30:8080   47m
```
- `app-backend`：代表正在對外提供服務的 stable 版本
- `app-backend-canary`：代表新部署的服務版本

另外也配置了兩個 Ingress
```shell
kubectl get ingress

NAME             CLASS    HOSTS              ADDRESS     PORTS   AGE
canary-ingress   <none>   day24.ithome.com   localhost   80      29m
stable-ingress   <none>   day24.ithome.com   localhost   80      29m
```
- `stable-ingress`：將流量導向 stable 版本的服務
- `canary-ingress`：將流量導向 canary 版本的服務

兩個都會監聽 `http://day24.ithome.com:30000` 的請求，根據流量控制規則將流量分配到不同的服務版本。

# 透過 Ingress 控制流量
來看一下這兩個 Ingress 的配置哪裡不同。

**stable-ingress**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: stable-ingress
spec:
  rules:
    - host: day24.ithome.com
      http:
        paths:
          - pathType: ImplementationSpecific
            backend:
              service:
                name: app-backend
                port:
                  number: 80
```

**canary-ingress**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: canary-ingress
  annotations:
    nginx.ingress.kubernetes.io/canary: "true" # 啟用 Canary
    nginx.ingress.kubernetes.io/canary-by-header: "Canary"       # 判斷 Header 的 Key Name
    nginx.ingress.kubernetes.io/canary-by-header-value: "true"   # Canary Header 的值必須是 true 才導流
spec:
  rules:
    - host: day24.ithome.com
      http:
        paths:
          - pathType: ImplementationSpecific
            backend:
              service:
                name: app-backend-canary
                port:
                  number: 80
```
除了導向的 Service 不同之外，控制流量的關鍵配置是以下三個 Annotation
- `nginx.ingress.kubernetes.io/canary`：啟用 Canary 功能
- `nginx.ingress.kubernetes.io/canary-by-header`：設置 Canary 請求需要有 Header 的 key 值為 `Canary`。
- `nginx.ingress.kubernetes.io/canary-by-header-value`：當 Header 中 Canary: `true` 時，請求會被導向 Canary 版本。

## 藍綠部署（Blue/Green）
透過這些 Annotation 控制流量，就能在新版本對外發佈之前，使用指定 Header 的方式，進行新版本的功能性驗證，在對外發佈新版本，來實現 **藍綠部署（Blue/Green）**。

### 調用 Stable 版本
使用 curl 調用服務
```shell 
# call stable version
curl day24.ithome.com:30000 --resolve day24.ithome.com:30000:127.0.0.1
# Output
Hello, ConfigMap from `Product` enviorment.
```

### 調用 Canary 版本
當加上指定 Header 後，流量會導向新版本服務
```shell 
# call canary version
curl day24.ithome.com:30000 --resolve day24.ithome.com:30000:127.0.0.1 -H "Canary: true"

# Output
Hello, ConfigMap from `Product` enviorment.(Canary version)
```

### 更新流量至新版本
當新版本已驗證成功時，只需更新 `stable-ingress` 的 `spec.rules[0].http.paths[0].backend.service` 到新版本的 Service，Nginx Ingress Controller 會將所有流量切換至新版本。

## 金絲雀部署（Canary）
我們添加一個 Annotation 到 `canary-ingrss` 上，將 50% 的流量導向 Canary 版本
```shell
`kubectl annotate ingress canary-ingress nginx.ingress.kubernetes.io/canary-weight="50"`
```
- `nginx.ingress.kubernetes.io/canary-weight`：指定幾％的流量請求會被導向 Canary 版本。

接著使用 `curl` 測試，不指定 Header，系統會隨機將約一半的流量導向新版本：
```shell
# 連打 10 次 curl
for i in {1..10}; do curl day24.ithome.com:30000 --resolve day24.ithome.com:30000:127.0.0.1; echo ""; done
# Output
Hello, ConfigMap from `Product` enviorment.
Hello, ConfigMap from `Product` enviorment.
Hello, ConfigMap from `Product` enviorment.(Canary version)
Hello, ConfigMap from `Product` enviorment.
Hello, ConfigMap from `Product` enviorment.(Canary version)
Hello, ConfigMap from `Product` enviorment.(Canary version)
Hello, ConfigMap from `Product` enviorment.
Hello, ConfigMap from `Product` enviorment.(Canary version)
Hello, ConfigMap from `Product` enviorment.(Canary version)
Hello, ConfigMap from `Product` enviorment.
```

透過百分比流量控制的方式，我們能依據新版本的運作狀況，逐步提高或減少 `nginx.ingress.kubernetes.io/canary-weight` 的值，實現 **金絲雀部署（Canary）**，以減少新版本發佈時的風險，防止故障大規模擴散。

# 小結
今天演練了如何使用 Ingress NGINX Controller 實現 Kubernetes 中的兩種部署策略：藍綠部署（Blue/Green） 和 金絲雀部署（Canary）。

但這個方案，要準備兩套 Deployment 與 Service，並且需要 Ingress 做許多操作，實務上仍有許多不便，明天會介紹一個專注於處理部署策略的 CNCF Project：[Argo Rollout](https://argo-rollouts.readthedocs.io/en/stable/)

# Refernce
- [Ingress NGINX Controller](https://github.com/kubernetes/ingress-nginx/tree/main)
- [Argo Rollout](https://argo-rollouts.readthedocs.io/en/stable/)