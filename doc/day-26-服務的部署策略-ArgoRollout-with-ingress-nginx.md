
# Day-26 服務的部署策略 - Argo Rollouts with Ingress Nginx Controller

# 前言
昨天介紹了 Argo Rollouts 的藍綠部署功能，今天將結合 Ingress NGINX Controller 進行**金絲雀部署（Canary）**，實現流量管理（Traffic Management）。  
> 📘 若不使用流量管理，金絲雀部署的流量控制會受到限制，詳情請參閱[官方文件](https://argoproj.github.io/argo-rollouts/features/canary/#overview)。


## 環境準備
- **安裝 Ingress NGINX Controller**

  安裝方式參考： Day24 / 服務的部署策略 - Ingress NGINX Controller

- **安裝 Argo Rollouts**

  安裝方式參考：Day25 / 服務的部署策略 - Argo Rollouts

# 部署 Demo 服務
執行以下指令，將此 [Github repo](https://github.com/YihongGao/iThome_30Day_2024/blob/main/resources/day26/apps/canary/deploy.yml) 的 Demo 服務部署到 `ithome` namespeace
```shell
kubectl create namespace ithome
kubectl apply -f https://raw.githubusercontent.com/YihongGao/iThome_30Day_2024/refs/heads/main/resources/day26/apps/canary/deploy.yml
```

部署後會有以下資源：
- 1 個 Rollout： `rollouts.argoproj.io/app-backend` 代表我們正在運行的服務。
- 2 個 Service： `service/app-backend-stable` 和`app-backend-preview`，分別對應穩定版本與新版本。
- 2 個 Ingress： 
  - `primary-ingress`：負責穩定版本的 Ingress
  - `app-backend-primary-ingress-canary`：由 Argo Rollouts 自動建立的 Ingress

接著，我們來查看 Rollout 有什麼不同。
> [Rollout 的 YAML](https://github.com/YihongGao/iThome_30Day_2024/blob/main/resources/day26/apps/canary/rollout.yml)的內容
```yaml
  strategy:
    canary:
      canaryService: app-backend-preview  # required
      stableService: app-backend-stable  # required
      trafficRouting:
        nginx:
          stableIngress: primary-ingress
          additionalIngressAnnotations:   # optional
            canary-by-header: Canary
            canary-by-header-value: "true"
      steps: # Deploy 
      - setWeight: 10
      - pause: {}
      - setWeight: 33
      - pause: {}
```
### 主要欄位介紹：
  - `stableService`、`canaryService`：分別用於穩定版本和新版本的 Pod，作為流量入口的 Kubernetes Service。
  - `trafficRouting`：定義 **Traffic Management** 配置
    - `nginx`：使用 Ingress NGINX controller 作為 **Traffic Management** 的實現
      - `stableIngress`：負責穩定版本的流量。
      - `additionalIngressAnnotations`：用來添加 NGINX Ingress 的 Annotation 
  - `steps`：定義多個發佈階段
    - `setWeight`：每個階段設定將多少百分比的流量導向新版本。
    - `pause`：在每個階段暫停，需手動 `Promote` 才能進入下一階段。

### 這個配置與藍綠部署相似，具備以下行為：
- 在完全切換到新版本之前，舊版本與新版本的 Pod 會同時運行。
- 透過 stableService 和 canaryService 分別導流到對應版本的 Pod。

此外，`trafficRouting.nginx` 允許 Argo Rollouts 自動管理 Nginx Ingress 流量轉導規則，省去自行管理多組 Ingress 的麻煩。

使用 `kubectl get ingress` 可以看到 Argo Rollouts 自動產生了一個 Ingress，其名稱格式為：`<ROLLOUT-NAME>-<INGRESS-NAME>-canary`
```shell
kubectl get ingress
NAME                                 CLASS    HOSTS              ADDRESS     PORTS   AGE
app-backend-primary-ingress-canary   <none>   day24.ithome.com   localhost   80      54m
primary-ingress                      <none>   day24.ithome.com   localhost   80      54m
```

另外我們可以使用 `steps` 來定義發布計畫，讓後續的版本發布能夠逐步自動或手動進行。
![https://argo-rollouts.readthedocs.io/en/stable/concepts-assets/canary-deployments.png](https://argo-rollouts.readthedocs.io/en/stable/concepts-assets/canary-deployments.png)

# 驗證 金絲雀部署（Canary）的行為
1. 模擬服務更新    
  使用 `kubectl edit` 更新 Rollout 資源的 Image Tag。
    ``` shell
    kubectl edit rollouts.argoproj.io/app-backend
    ```
    在編輯視窗中，將 spec.template.spec.containers[0].image 更新為：
    ```yaml
    spec:
      template:
        spec:
          containers:
          - image: luciferstut/app-backend-for-ithome2024:day-21-canary
    ```
2. 觀察 Pod, Service 的變化   
  **Pod**    
  新舊版本的 Pod 會同時運行，且每個 Pod 會有該版本專屬的 `rollouts-pod-template-hash` Label 
    ```shell
    kubectl get pod --show-labels -o wide
    NAME                           READY   STATUS    RESTARTS   AGE   IP            NODE                 NOMINATED NODE   READINESS GATES   LABELS
    app-backend-5cccbf7f97-ch6wc   1/1     Running   0          78s   10.244.0.46   kind-control-plane   <none>           <none>            app=app-backend,rollouts-pod-template-hash=5cccbf7f97
    app-backend-6c8c946c5-5kfp8    1/1     Running   0          51m   10.244.0.45   kind-control-plane   <none>           <none>            app=app-backend,rollouts-pod-template-hash=6c8c946c5
    ```
    
    **Service**    
    與藍綠部署相似，能看到 `app-backend-stable`/`app-backend-preview` 分別作為 穩定/新版本的流量入口。
    ```shell
    kubectl get svc,ep -o wide

    NAME                          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE   SELECTOR
    service/app-backend-preview   ClusterIP   10.96.68.31    <none>        80/TCP    76m   app=app-backend,rollouts-pod-template-hash=5cccbf7f97
    service/app-backend-stable    ClusterIP   10.96.254.87   <none>        80/TCP    76m   app=app-backend,rollouts-pod-template-hash=6c8c946c5

    NAME                            ENDPOINTS          AGE
    endpoints/app-backend-preview   10.244.0.46:8080   76m
    endpoints/app-backend-stable    10.244.0.45:8080   76m
    ```
3. 觀察 Ingress 
  透過 `kubectl get ingress app-backend-primary-ingress-canary -o yaml` 檢視 `app-backend-primary-ingress-canary` 的內容
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    annotations:
      nginx.ingress.kubernetes.io/canary: "true" # Argo Rollouts 會自動配置此 Annotation 自動開啟 Canary 功能
      nginx.ingress.kubernetes.io/canary-by-header: Canary # 依照 Rollout YAML 中 additionalIngressAnnotations 定義添加的 Annotation
      nginx.ingress.kubernetes.io/canary-by-header-value: "true" # 依照 Rollout YAML 中 additionalIngressAnnotations 定義添加的 Annotation
      nginx.ingress.kubernetes.io/canary-weight: "10" # 依照 Rollout YAML 中 steps.setWeight 配置，依據每個階段配置流量權重
    name: app-backend-primary-ingress-canary
    # ... 省略
  spec:
    rules:
    - host: day24.ithome.com
      http:
        paths:
        - backend:
            service:
              name: app-backend-preview
              port:
                number: 80
          pathType: ImplementationSpecific
  ```
  我們能看到許多 Day24 介紹中用來控制流量的 Nginx Annotation，當 Argo Rollouts 使用 NGINX 作為 **Traffic Management** 時，就是透過這些 Annotation 進行流量控制，而我們只需在 Rollout YAML 中定義好發佈計畫即可，無需手動管理多組 Ingress 配置。
      
# 逐步開放流量
開啟 [Argo Rollouts Dashboard](https://argoproj.github.io/argo-rollouts/dashboard/) 能看到這個 Rollout 目前的發布了 10% 流量到新版本
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240927/截圖-2024-09-27-下午10.25.13.6m3vbukv8z.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240927/截圖-2024-09-27-下午10.25.13.6m3vbukv8z.webp)

檢視 Canary Ingress 流量權重的 Annotation
```shell
kubectl get ingress app-backend-primary-ingress-canary -o yaml |grep weight

# Output
    nginx.ingress.kubernetes.io/canary-weight: "10"
```

能看出 Ingress 配置與 steps 定義相符，接著使用 `curl` 驗證看看，應該約 10% 機率會有 Canary version 的 Response
```shell
for i in {1..10}; do curl day24.ithome.com:30000 --resolve day24.ithome.com:30000:127.0.0.1; echo ""; done

# Output
Hello, welcome to use the container.(Canary version)
Hello, welcome to use the container.
Hello, welcome to use the container.
Hello, welcome to use the container.
Hello, welcome to use the container.
Hello, welcome to use the container.
Hello, welcome to use the container.
Hello, welcome to use the container.
Hello, welcome to use the container.
Hello, welcome to use the container.
```

接下來，我們透過 CLI 或 UI 進行 `Promote` 讓發布進入到下一個階段：33% 流量到
```shell
kubectl argo rollouts promote app-backend               
```

再次檢視 Canary Ingress 配置
```shell
 k get ingress app-backend-primary-ingress-canary -o yaml |grep weight
    nginx.ingress.kubernetes.io/canary-weight: "33"
```
能發現，我們無需像 Day24 的 Demo 一樣手動調整 Canary Ingress 配置，Argo Rollouts 會根據 steps 設定自動管理流量切換。

再次透過 `curl` 驗證時，應能感受到 Canary Response 的頻率變高了
```shell
Hello, welcome to use the container.
Hello, welcome to use the container.
Hello, welcome to use the container.(Canary version)
Hello, welcome to use the container.
Hello, welcome to use the container.(Canary version)
Hello, welcome to use the container.
Hello, welcome to use the container.
Hello, welcome to use the container.
Hello, welcome to use the container.
Hello, welcome to use the container.(Canary version)
```

我們在進行一次 `Promote` 進入全量切換
```shell
kubectl argo rollouts promote app-backend  
```
檢查 Ingess 配置
```shell
kubectl get ingress app-backend-primary-ingress-canary -o yaml |grep weight
    nginx.ingress.kubernetes.io/canary-weight: "0"
```
會發現 `canary-weight` 已被設置為 0，因為全量切換後，新版本的 Pod 被提升為穩定版本，舊版本的 Pod 會自動關閉，因此不再透過 Canary Ingress 傳遞流量。

# 小結
今天我們介紹了如何使用 Argo Rollouts 與 Ingress NGINX Controller 實現金絲雀部署，實現了逐步控制流量導向新版本應用的能力。

借助 Argo Rollouts 的流量管理功能，使用者可以輕鬆配置不同的發佈階段，無需手動管理多組 Ingress 設定。這種方式簡化了流量控制的複雜度，並提升了應用更新過程中的靈活性與穩定性。

# Refernce
- [Ingress NGINX Controller](https://github.com/kubernetes/ingress-nginx/tree/main)
- [Argo Rollout](https://argo-rollouts.readthedocs.io/en/stable/)