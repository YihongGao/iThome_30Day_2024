# Day-19 ArgoCD 介紹

# 前言
去年，我透過 GitLab CI 實現了 Push-based 的 GitOps，能自動將 Git Repo 中的 YAML 部署到 Kubernetes。
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午2.39.33.26lfru5jsv.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午2.39.33.26lfru5jsv.webp)

雖然 Push 模式實現簡單且架構輕量，但仍有幾個問題需要改善：
1. 安全性風險：Push 模式需要暴露 Cluster 的 API server，增加了公開網絡中的安全風險。
2. 難以擴展：隨著 Cluster 數量增加，Push 模式在多 Cluster 管理，特別是跨網域時，變得困難。
3. 一致性問題：如果有人為操作 Kubernetes，可能導致 Cluster 與 Git Repo 狀態不同步

3. 一致性問題：如果有人為操作 Kubernetes，可能導致 Cluster 與 Git Repo 狀態不同步

# 什麼是 ArgoCD
![https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcT5JxBwyzDrpsoZJboHIdNCwZMma8GGgQ1uuQ&s](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcT5JxBwyzDrpsoZJboHIdNCwZMma8GGgQ1uuQ&s)

ArgoCD 是一款基於 Pull-based GitOps 模型的 Kubernetes 持續交付工具。當 Git Repo 中的 YAML 文件更新時，ArgoCD 會自動將這些變更同步到 Kubernetes，並持續監控 Kubernetes 與 Git Repo 的狀態，確保雙方狀態一致。

![https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午2.39.43.8s39iv9lry.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午2.39.43.8s39iv9lry.webp)

ArgoCD 提供了豐富的開箱即用功能：
- **CLI 與 Web UI**：有 UI 就給讚
- **Webhook**：能依據需求進行各種串接
- **Notification**：輕鬆整合 Slack 等通訊軟體，輕鬆實現告警
- **OAuth**：能與 Keycloak、Google 等整合，不需額外管理一套使用者帳號

廢話不多說，我們直接來體驗一下 ArgoCD

# 環境準備

## 安裝 ArgoCD
根據[官方安裝文件]，使用 Kustomize 的 Remote Resource 安裝 ArgoCD
```yaml
# argoCD-demo/infra/argoCD/kustomization.yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: argocd
resources:
- https://raw.githubusercontent.com/argoproj/argo-cd/v2.7.2/manifests/install.yaml
```
接著將 ArgoCD 部署到 Kubernetes 中：
```shell
# pwd
# /day19/argoCD-demo <- current folder

# install
kubectl create namespace argocd
kubectl apply -k infra/argoCD
```

## 安裝 ArgoCD CLI
Mac 可以直接透過 Homebrew 安裝，其他平台的安裝方法可參考 [官方文件](https://argo-cd.readthedocs.io/en/stable/cli_installation/#installation)
```shell
# Mac
brew install argocd

# check the install is successful
argocd version
```
## 登入 Web UI
ArgoCD 的 Web UI 運行於 Kubernetes 的 Pod 中，使用 `kubectl port-forward`  將 Web 服務轉發到本地的 8080 端口，方便 Demo 存取：
```shell
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
ArgoCD 的 Web UI 運行於 Kubernetes 的 Pod 中，使用 `kubectl port-forward`  將 Web 服務轉發到本地的 8080 端口，方便 Demo 存取：
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午3.58.49.6m3ux6bjew.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午3.58.49.6m3ux6bjew.webp)

透過 `argocd` CLI，取得初始化密碼
```shell
argocd admin initial-password -n argocd

# output
w4sgE3j7Mb97NEzB

 This password must be only used for first time login. We strongly recommend you update the password using `argocd account update-password`.
```
> ⚠️ 此密碼僅用於首次登入。建議讀者參閱 [官方文件](https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli) 來修改密碼。

使用 `admin` 帳號與剛取得的密碼登入
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午4.07.34.4xuhzzwl9n.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午4.07.34.4xuhzzwl9n.webp)

## 透過 ArgoCD 部署服務到 Kubernetes 
我們可以使用前幾天的 Kustomize 配置來部署服務。在此之前，先清理環境並重新創建 ithome namespace，以便觀察 ArgoCD 的部署行為：
```shell
# 環境清理
kubectl delete namespace ithome
kubectl create namespace ithome
```

1. 建立 Apps
點選 **NEW APP** 按鈕
![https://argo-cd.readthedocs.io/en/stable/assets/new-app.png](https://argo-cd.readthedocs.io/en/stable/assets/new-app.png)

2. 填寫 Application 基本資料
- **Application**：`argocd-demo`
- **Project**：`default`
- **Sync Policy**：`Automatic` 並勾選 `SELF HEAL`
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午4.23.43.8l01njdag9.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午4.23.43.8l01njdag9.webp)

3. 填寫 Git Repo 資訊   
  使用此 [Github Repo](https://github.com/YihongGao/iThome_30Day_2024) 作為 YAML 的來源，並將其中的資源部署至 Kubernetes。

- **Repository URL**：`https://github.com/YihongGao/iThome_30Day_2024`
- **Revision**：`main`
- **Path**：`resources/day19/argoCD-demo/apps/overlays/production`
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午4.31.04.8ojnl9h5vw.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午4.31.04.8ojnl9h5vw.webp)

4. 配置要部署到哪個 Kubernetes
- **Cluster URL**：`https://kubernetes.default.svc`（代表部署到與 ArgoCD 同一 Kubernetes Cluster）
- **Namespace**：`ithome`
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午4.35.45.1e8ka7xo6f.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午4.35.45.1e8ka7xo6f.webp)

  全部填寫完成後，點選上方 **CREATE** 按鈕
  ![https://argo-cd.readthedocs.io/en/stable/assets/create-app.png](https://argo-cd.readthedocs.io/en/stable/assets/create-app.png)

  到這裡，我們已經在 ArgoCD 上配置了
  - 使用哪個 Git Repo 作為 Kubernetes 期望狀態
  - 要將 YAML 部署到哪個 Kubernetes Cluster 與 namespace
  
  在 UI 介面上，能看到名為 `argocd-demo` 的 Application
  ![https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午4.46.42.8hgfpudlgh.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午4.46.42.8hgfpudlgh.webp)


## Application 狀態解釋
  能看到 `argocd-demo` Application 的 Status 欄位，可以看到兩個主要狀態：
  1. **App Health** 表示 ArgoCD 管理的 Kubernetes 資源運行情況，主要有以下幾個狀態：    
    - **Healthy**：所有資源運行正常，並通過健康檢查（如 Pod 通過 liveness 和 readiness 檢查）。   
    - **Progressing**：資源正在部署或更新中，部分資源尚未完全啟動或達到健康狀態。   
    - **Degraded**：部分資源運行異常，可能有 Pod 啟動失敗或 CrashLoopBackOff 等問題。   

  2. Sync Status：表示 Application 的狀態是否與 Git Repo 同步：    
    - **Synced（已同步）**：Kubernetes 狀態與 Git Repo 完全一致，所有變更已成功應用到 Cluster。   
    - **OutOfSync（不同步）**：Application 的狀態與 Git Repo 不一致，可能是變更尚未同步到 Cluster 或部分資源不符預期。   

## 驗證部署狀態
  我們使用 kubectl 來檢查 Kubernetes 是否如 App Status 所示運作正常：
  ```shell
  kubectl get pod

  # output
  NAME                                READY   STATUS    RESTARTS   AGE
  app-backend-7b8d5c4cd7-ntkhw        1/1     Running   0          2m35s
  app-backend-7b8d5c4cd7-rnxfs        1/1     Running   0          2m24s
  product-backend-5564f9975c-8c4k9    1/1     Running   0          2m9s
  product-backend-5564f9975c-8m2lg    1/1     Running   0          2m30s
  product-schedule-7ccf4f66ff-vfs27   1/1     Running   0          2m27s
  ```
  可以看到，Git Repo 中的 YAML 已成功部署到 Kubernetes。接下來，我們嘗試刪除一個 Resource（例如 `Deployment`）來測試自動恢復功能：
  ```yaml
  kubectl delete deployment app-backend
  ```
  > 📘 若沒有自動建立回來，請檢查 `argocd-demo` 中的 **SYNC POLICY** 是否有 Enable `AUTOMATED` 與 `SELF HEAL`

  這時可以看到該 Deployment 會自動被重建。這正是 Pull-based GitOps 的優勢所在：
  - **ArgoCD(Pull-based)**：持續監控並自動修復，確保 Kubernetes 狀態與 Git Repo 宣告一致。
  - **Push-based**：依賴頻繁的發佈與嚴謹的工作流程來維持狀態同步。

# 半自動化的方式
如果讀者的公司規範不適合使用全自動化作業，ArgoCD 也支援半自動化模式，並提供直觀的 Diff 介面來檢視 Kubernetes 與 Git Repo 之間的差異。

讀者能透過以下操作體驗半自動化的流程
1. Disable `SELF HEAL`
2. 更改任一資源，例如我更改 Deployment imageTag
    ```shell
    # 從 1.0 改為 latest
    kubectl set image deployment/app-backend apps=luciferstut/app-backend-for-ithome2024:latest
    ```
3. 不久後，argocd-demo 的 Sync status 應該會轉為 `OutOfSync`
  ![https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-18-上午12.47.47.4ckue7n0fz.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-18-上午12.47.47.4ckue7n0fz.webp)
4. 點選下方 app-backend
  ![https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-18-上午12.49.22.99tb823atw.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-18-上午12.49.22.99tb823atw.webp)
5. 再展開的頁面下方有個 Diff 頁籤，能看到 Git Repo 與 Kubernetes 的當前狀態差異。
  ![https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-18-上午12.49.41.7ljyavd0nn.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-18-上午12.49.41.7ljyavd0nn.webp)
6.當檢視完差異後，若判斷應該要同步到 Kubernetes 時，點選 **SYNC**，就能將 Git Repo 的 YAML 部署到 Kubernetes
  ![https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-18-上午12.51.35.6t72t4yumy.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-18-上午12.51.35.6t72t4yumy.webp)
  

# 小結
今天我們初步體驗了 ArgoCD 的功能，特別是其自動保持 Git Repo 與 Kubernetes 狀態一致的能力。這大幅提升了在使用 GitOps 管理 Kubernetes 時的信心和可維護性，避免 Git Repo 與 Kubernetes 狀態脫鉤太久，導致操作風險上升。

明天，我們將深入介紹 ArgoCD 的架構與運作原理。

# Refernce
- [ArgoCD 官方文件](https://argo-cd.readthedocs.io/en/stable/)


[官方安裝文件]: https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/#kustomize