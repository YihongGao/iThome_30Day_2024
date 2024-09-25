
# Day-25 服務的部署策略 - Argo Rollouts

# 前言
昨天透過 Ingress NGINX controller 實作了
  - 藍綠部署（Blue/Green）
  - 金絲雀部署（Canary）

但其中控制流量的手續繁瑣，而且每次需要多部署一份新版本的 Deployment、Service 再管理上十分不方便。

今天來介紹 CNCF 中專注在部署策略解決方案的專案：**Argo Rollouts**

# Argo Rollouts
![https://miro.medium.com/v2/resize:fit:1400/1*rZ_Yfz9XNk8dDqf4s8kDhQ.jpeg](https://miro.medium.com/v2/resize:fit:1400/1*rZ_Yfz9XNk8dDqf4s8kDhQ.jpeg)
Argo Rollouts 是一個漸進式部署（Progressive Delivery）到 Kubernetes 的解決方案，透過 Argo Rollouts 提供的 CustomResourceDefinition (CRD) 能輕鬆的使用 **金絲雀部署（Canary）**、**藍綠部署（Blue/Green)** 來發布 Kubernetes 的 workload。

# Argo Rollouts 架構
![https://argo-rollouts.readthedocs.io/en/stable/architecture-assets/argo-rollout-architecture.png](https://argo-rollouts.readthedocs.io/en/stable/architecture-assets/argo-rollout-architecture.png)

主要的架構組件：
- **Custom Resource Definitions (CRD)**：
  - **Rollout**：Argo Rollouts 的核心 CRD，用來替代標準的 Kubernetes Deployment，能定義複雜的部署策略（如金絲雀部署和藍綠部署）。
  - **AnalysisTemplate** 和 **AnalysisRun**：分別用於定義和執行分析任務。這些資源可以透過 metrics 或外部監控系統（如 Prometheus）來判斷新版本的健康狀態

- **Rollout Controller**：監控 Rollout 資源的狀態，根據定義的策略自動管理流量和部署過程，並負責健康檢查和 Rollback。

- **Traffic Management**：處理流量控制的組件，可選擇許多不同的組件來實現流量管理，提供 Argo Rollouts 精準控制流量到新舊版本 Pod 的能力

直接來安裝並體驗看看使用上是什麼感覺

# 環境準備
1. 安裝 Ingress NGINX Controller 到本地 kind 環境
    > 若於 Day24 安裝過 Ingress NGINX Controller，可跳過此步驟
    
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
2. 安裝 Argo Rollouts 到 Kubernetes  

參考 [官方文件](https://argoproj.github.io/argo-rollouts/installation/#controller-installation) 執行以下安裝指令
```shell
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```
執行完成後，會建立一個 namespace：argo-rollouts 並運行 Argo Rollouts controller。
```shell
kubectl get deployments.apps,pod -n argo-rollouts 
```
2. 安裝 Argo Rollout Kubectl Plugin
透過 Brew 安裝或參考 [官方文件其他安裝方式](https://argoproj.github.io/argo-rollouts/installation/#kubectl-plugin-installation)
```shell
brew install argoproj/tap/kubectl-argo-rollouts
```
檢查安裝是否成功
```shell
kubectl argo rollouts version

# Output
kubectl-argo-rollouts: v1.7.2+59e5bd3
BuildDate: 2024-08-13T18:29:47Z
GitCommit: 59e5bd385c031600f86075beb9d77620f8d7915e
GitTreeState: clean
GoVersion: go1.21.13
Compiler: gc
Platform: darwin/amd64
```

# 部署 Demo 服務
執行以下指令，將此 [Github repo](https://github.com/YihongGao/iThome_30Day_2024/tree/main/resources/day25/apps/blue-green) 的 Demo 服務部署到 `ithome` namespeace
```shell
kubectl create namespace ithome
kubectl apply -f https://raw.githubusercontent.com/YihongGao/iThome_30Day_2024/refs/heads/main/resources/day25/apps/blue-green/deploy.yaml
```

部署後會有下資源
- `rollouts.argoproj.io/app-backend`：代表我們運行的服務
```shell
kubectl  get rollouts.argoproj.io,pod -o wide

# Output
NAME                              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
rollout.argoproj.io/app-backend   1         1         1            1           17m

NAME                              READY   STATUS    RESTARTS   AGE   IP            NODE                 NOMINATED NODE   READINESS GATES
pod/app-backend-6c8c946c5-6dlvx   1/1     Running   0          17m   10.244.0.41   kind-control-plane   <none>           <none>
```
檢視 Rollout 的 [YAML](https://github.com/YihongGao/iThome_30Day_2024/blob/main/resources/day25/apps/blue-green/deployment.yml) 能看到大部分的屬性都跟 Deployment 很相似，他們的確也都具有相同的功能性，主要的差異再 `strategy`，定義了藍綠部署的配置
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: app-backend
spec:
  ...
  strategy:
    blueGreen: 
      # activeService specifies the service to update with the new template hash at time of promotion.
      # This field is mandatory for the blueGreen update strategy.
      activeService: app-backend-stable
      # previewService specifies the service to update with the new template hash before promotion.
      # This allows the preview stack to be reachable without serving production traffic.
      # This field is optional.
      previewService: app-backend-preview
      # autoPromotionEnabled disables automated promotion of the new stack by pausing the rollout
      # immediately before the promotion. If omitted, the default behavior is to promote the new
      # stack as soon as the ReplicaSet are completely ready/available.
      # Rollouts can be resumed using: `kubectl argo rollouts promote ROLLOUT`
      autoPromotionEnabled: false
```
依此為例
- `strategy`：部署策略的定義
  - `blueGreen`：使用藍綠部署
      - `activeService`：穩定版本的 Pod 要透過哪個 Kubernetes Service 當流量入口
      - `previewService`：新版本的 Pod 要透過哪個 Kubernetes Service 當流量入口
      - `autoPromotionEnabled`：當新版本的 Pod Ready 時，是否要自動執行流量切換，並取代及關閉穩定版本的 Pod

當這個 Rollout 版本更新時，會同時運行新舊版本的 Pod，並且能透過 `activeService`、`previewService` 定義的 Service 向指定版本的 Pod 傳遞流量。

- `service/app-backend-stable`、`app-backend-preview`：分別用來作為服務新舊版本的流量入口。能發現目前都是導向同一個 Pod，當新版本出現時，會 `app-backend-preview` 會自動調整流量轉發到新版本 Pod 而不是當前版本。
```shell
kubectl get svc,ep

# Output
NAME                          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/app-backend-preview   ClusterIP   10.96.132.143   <none>        80/TCP    18m
service/app-backend-stable    ClusterIP   10.96.199.44    <none>        80/TCP    18m

NAME                            ENDPOINTS          AGE
endpoints/app-backend-preview   10.244.0.41:8080   18m
endpoints/app-backend-stable    10.244.0.41:8080   18m
```

-`ingress/stable-ingress`、`ingress/canary-ingress`：與 Day24 相同，僅是個單純的 Ingress，會監聽 `http://day24.ithome.com:30000` ，並依照流量控制將流量轉發到不同版本的服務。

```shell
kubectl get ingress

# Output
NAME             CLASS    HOSTS              ADDRESS     PORTS   AGE
canary-ingress   <none>   day24.ithome.com   localhost   80      17m
stable-ingress   <none>   day24.ithome.com   localhost   80      18m
```

驗證一下 Demo 服務是否運作正常
```shell
# 預設請求會打向 stable-ingress
curl day24.ithome.com:30000 --resolve day24.ithome.com:30000:127.0.0.1

# Output，返回 stable 版本的 reponse
Hello, welcome to use the container.

# 透過 Header 指定打向 canary-ingress
curl day24.ithome.com:30000 --resolve day24.ithome.com:30000:127.0.0.1 -H "Canary: true"

# Output，與 stable 一致，因為目前只有運行 stable 版本的 Workload
Hello, welcome to use the container.
```

# 驗證 藍綠部署（Blue/Green）的行為
1. 模擬服務更新    
  更新 Rollout 使用的 Image Tag，使用 `kubectl edit` 編輯 `rollouts.argoproj.io/app-backend`
    ``` shell
    kubectl edit rollouts.argoproj.io/app-backend
    ```
    更新 spec.template.spec.containers[0].image 的值為 `luciferstut/app-backend-for-ithome2024:day-21-canary`
    ```yaml
    spec:
      template:
        spec:
          containers:
          - image: luciferstut/app-backend-for-ithome2024:day-21-canary
    ```
2. 觀察 Pod, Service 有什麼改變    
  **Pod**    
  能看到出現了一個使用新版本 Image 的 Pod，並且舊版本的 Pod 仍在運作。
    ```shell
    kubectl get pod --show-labels -o wide

    # Output
    NAME                           READY   STATUS    RESTARTS   AGE   IP            NODE                 NOMINATED NODE   READINESS GATES   LABELS
    app-backend-5cccbf7f97-nzsm9   1/1     Running   0          18m   10.244.0.42   kind-control-plane   <none>           <none>            app=app-backend,rollouts-pod-template-hash=5cccbf7f97
    app-backend-6c8c946c5-6dlvx    1/1     Running   0          66m   10.244.0.41   kind-control-plane   <none>           <none>            app=app-backend,rollouts-pod-template-hash=6c8c946c5
    ```
    不同版本的 Pod 分別有不同的 `rollouts-pod-template-hash` Label value。

  **Service**    
  這時能看到 stable 與 preview 的 Service 分別透過 Label Selector 與 `rollouts-pod-template-hash` 流量分配到對應版本的 Pod
  ```shell
  kubectl get svc,ep -o wide

  # Output
  NAME                          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE   SELECTOR
  service/app-backend-preview   ClusterIP   10.96.132.143   <none>        80/TCP    65m   app=app-backend,rollouts-pod-template-hash=5cccbf7f97
  service/app-backend-stable    ClusterIP   10.96.199.44    <none>        80/TCP    65m   app=app-backend,rollouts-pod-template-hash=6c8c946c5

  NAME                            ENDPOINTS          AGE
  endpoints/app-backend-preview   10.244.0.42:8080   65m
  endpoints/app-backend-stable    10.244.0.41:8080   65m
  ```

  到這裡，我們透過 Argo Rollouts 來同時運行新舊版本的 Pod 並且會自動調整 Service 的 Label Selector，我們不在需要管理多個 Deployment。

# 透過 Ingress 驗證流量是否正確分派到新舊版本
接下來只要調用正確的 Service 就能將流量往該版本發送，這裡依據 Ingress 的配置來發送流量給對應的 Service
```shell
# 預設請求會打向 stable-ingress
curl day24.ithome.com:30000 --resolve day24.ithome.com:30000:127.0.0.1

# Output，返回 stable 版本的 Reponse
Hello, welcome to use the container.

# 透過 Header 指定打向 canary-ingress
curl day24.ithome.com:30000 --resolve day24.ithome.com:30000:127.0.0.1 -H "Canary: true"

# Output，Canary 版本的 Response
Hello, welcome to use the container.(Canary version).
```
能看到分別回應了新舊版本的 Response。

# 全量切換
當藍綠部署（Blue/Green）對新版本的服務驗證完成後，要將流量都轉向新版本服務，並且關閉舊版本的服務，來看看 Argo Rollouts 怎麼做

1. 透過 CLI 執行 `Promote` 完成全量切換
```shell
kubectl argo rollouts promote app-backend --full
```

2. 或透過 Argo Rollouts Dashboard 進行 `promote`
轉發 Dashboard 到 `http://localhost:3100`
```shell
kubectl argo rollouts dashboard
```
用瀏覽器開啟 Dashboard，並點選該服務的 `Promote` 按鈕來完成全量切換
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240926/截圖-2024-09-25-下午11.58.48.26lg3tpg5j.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240926/截圖-2024-09-25-下午11.58.48.26lg3tpg5j.webp)

當透過 CLI 或 Dashboard 執行 `Promote` 後，能發現舊版本的 Pod 都被關閉了，而不管是 stable 還是 preview 的 Service 也都會指向新版本的 Pod，不管連到任何一個 Service 都不會有連不到服務的問題。

# 小結


# Refernce
- [Ingress NGINX Controller](https://github.com/kubernetes/ingress-nginx/tree/main)
- [Argo Rollout](https://argo-rollouts.readthedocs.io/en/stable/)