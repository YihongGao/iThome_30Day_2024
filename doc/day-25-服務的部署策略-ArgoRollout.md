
# Day-25 服務的部署策略 - Argo Rollouts

# 前言
昨天使用 Ingress NGINX Controller 實作了以下部署策略：
  - 藍綠部署（Blue/Green）
  - 金絲雀部署（Canary）

但在操作中，控制流量的過程相當繁瑣。每次發佈新版本都需要額外部署新的 Deployment 和 Service，這在管理上變得不便。

今天要介紹 CNCF 專注於部署策略的解決方案：**Argo Rollouts**。

# Argo Rollouts
![https://miro.medium.com/v2/resize:fit:1400/1*rZ_Yfz9XNk8dDqf4s8kDhQ.jpeg](https://miro.medium.com/v2/resize:fit:1400/1*rZ_Yfz9XNk8dDqf4s8kDhQ.jpeg)

**Argo Rollouts** 是一個針對 Kubernetes 的漸進式部署（Progressive Delivery）解決方案。它提供了CustomResourceDefinition（CRD），使我們能夠輕鬆實現 **金絲雀部署（Canary）** 和 **藍綠部署（Blue/Green）** 等進階的發布策略，優雅地管理 Kubernetes workload。

# Argo Rollouts 架構
![https://argo-rollouts.readthedocs.io/en/stable/architecture-assets/argo-rollout-architecture.png](https://argo-rollouts.readthedocs.io/en/stable/architecture-assets/argo-rollout-architecture.png)

主要的架構組件：
- **Custom Resource Definitions (CRD)**：
  - **Rollout**：Argo Rollouts 的核心 CRD，用來替代標準的 Kubernetes Deployment，能定義複雜的部署策略（如金絲雀部署和藍綠部署）。
  - **AnalysisTemplate** 和 **AnalysisRun**：分別用於定義和執行分析任務。這些資源可以透過 metrics 或外部監控系統（如 Prometheus）來判斷新版本的健康狀態

- **Rollout Controller**：監控 Rollout 資源的狀態，根據定義的策略自動管理流量和部署過程，並負責健康檢查和 Rollback。

- **Traffic Management**：處理流量控制的組件，可選擇許多不同的組件來實現流量管理，提供 Argo Rollouts 精準控制流量到新舊版本 Pod 的能力

現在就來安裝 Argo Rollouts 並實 際體驗看看他的能力

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

    依據 [官方文件](https://argoproj.github.io/argo-rollouts/installation/#controller-installation) 執行以下安裝指令
    ```shell
    kubectl create namespace argo-rollouts
    kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
    ```
    執行完成後，會建立一個 namespace：argo-rollouts 並運行 Argo Rollouts controller。
    ```shell
    kubectl get deployments.apps,pod -n argo-rollouts 
    ```
2. 安裝 Argo Rollout Kubectl Plugin
    Mac 能透過 Brew 安裝 或 參考 [官方文件其他安裝方式](https://argoproj.github.io/argo-rollouts/installation/#kubectl-plugin-installation)
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

部署後會有以下資源：
- 1 個 Rollout： `rollouts.argoproj.io/app-backend` 代表我們正在運行的服務。
- 2 個 Service： `service/app-backend-stable` 和`app-backend-preview`，分別對應穩定版本與新版本。
- 2 個 Ingress： 
  - `primary-ingress`：負責穩定版本的 Ingress
  - `canary-ingress`：負責新版本版本的 Ingress

能使用 `kubectl get rollouts.argoproj.io` 檢視 Rollout 資源
  ```shell
  kubectl get rollouts.argoproj.io,pod -o wide

  # Output
  NAME                              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
  rollout.argoproj.io/app-backend   1         1         1            1           17m

  NAME                              READY   STATUS    RESTARTS   AGE   IP            NODE                 NOMINATED NODE   READINESS GATES
  pod/app-backend-6c8c946c5-6dlvx   1/1     Running   0          17m   10.244.0.41   kind-control-plane   <none>           <none>
  ```
  Rollout 資源能理解為 Deployment 的子類別，擁有更多的部署策略可使用。所以檢視 [Rollout 的 YAML](https://github.com/YihongGao/iThome_30Day_2024/blob/main/resources/day25/apps/blue-green/rollout.yml)時，能發現大部分屬性與 Deployment 相同，因為兩者具有相同的核心功能。

  當要使用進階的部署策略時，可透過strategy欄位進行配置，如此例使用 **藍綠部署（Blue/Green）**
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
  - `strategy`：定義部署策略。
    - `blueGreen`：使用藍綠部署模式
        - `activeService`：用於穩定版本的 Pod，作為流量入口的 Kubernetes Service。
        - `previewService`：用於新版本的 Pod，作為流量入口的 Kubernetes Service。
        - `autoPromotionEnabled`：控制當新版本的 Pod 準備就緒時，是否自動切換流量，並關閉舊版本的 Pod。

  在 Rollout 更新時，會同時運行新舊版本的 Pod，並透過 `activeService` 和 `previewService` 將流量導向相應的版本。

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
  能看到新版本的 Pod 啟動，舊版本的 Pod 依然運行：
    ```shell
    kubectl get pod --show-labels -o wide

    # Output
    NAME                           READY   STATUS    RESTARTS   AGE   IP            NODE                 NOMINATED NODE   READINESS GATES   LABELS
    app-backend-5cccbf7f97-nzsm9   1/1     Running   0          18m   10.244.0.42   kind-control-plane   <none>           <none>            app=app-backend,rollouts-pod-template-hash=5cccbf7f97
    app-backend-6c8c946c5-6dlvx    1/1     Running   0          66m   10.244.0.41   kind-control-plane   <none>           <none>            app=app-backend,rollouts-pod-template-hash=6c8c946c5
    ```
    不同版本的 Pod 具有不同的 `rollouts-pod-template-hash` Label value。
    
    **Service**    
    stable 和 preview 兩個 Service 根據 `rollouts-pod-template-hash` 將流量導向對應的 Pod：
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
    此時，我們可以看到 `app-backend-preview` Service 導向新版本的 Pod，而 `app-backend-stable` 繼續導向舊版本的 Pod

    通過 Argo Rollouts，我們可以同時運行新舊版本的 Pod，並自動調整 Service 的 Label Selector，不再需要手動管理多個 Deployment。

# 透過 Ingress 驗證流量是否正確分派到新舊版本
接下來，我們將透過 Ingress 來驗證流量是否正確分派到不同版本的 Service。
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
從輸出的結果可以看到，根據不同的 Ingress 配置，流量被正確分派到新舊版本的服務。

# 全量切換
當藍綠部署（Blue/Green）驗證新版本服務沒有問題後，將所有流量切換到新版本服務，並關閉舊版本的服務。這可以透過 Argo Rollouts 輕鬆完成。

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

不論是透過 CLI 還是 Dashboard 進行全量切換，切換完成後可以觀察到舊版本的 Pod 被自動關閉，stable 和 preview 這兩個 Service 會統一指向新版本的 Pod。這樣確保了無論使用哪個 Service，流量都會正確轉發，不會出現連線錯誤。

# 小結
今天初步認識了 Argo Rollouts 的核心概念與使用方式，透過使用 Rollouts 的 CRD 取代 Deployment 當 workload，能輕鬆的使用漸進式部署（Progressive Delivery）不需要管理大量 workload YAML。

明天會將 Ingress NGINX Controller 作為 **Traffic Management** 與 Argo Rollouts 協作，簡化部署時，對 Ingress 的操作。

# Refernce
- [Ingress NGINX Controller](https://github.com/kubernetes/ingress-nginx/tree/main)
- [Argo Rollout](https://argo-rollouts.readthedocs.io/en/stable/)