
# Day-19 ArgoCD 介紹

# 前言
去年我透過 GitLab CI 實現了 Push base 的 GitOps，能自動將 Git Repo 中的 YAML 部署至 Kubernetes
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午2.39.33.26lfru5jsv.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午2.39.33.26lfru5jsv.webp)

雖然實現容易且架構簡單，但仍有許多值得改善的地方，例如
1. 安全性風險：Push 模式需要暴露 cluster 的 API server，增加公開網絡中的風險。
2. 難以擴展：隨著集群數量增加，Push 模式在多個 cluster 的管理上變得困難，尤其是跨網域時。
3. 一致性問題：若有人為操作 Kubernetes，可能導致 cluster 與 Git Repo 不同步。

我們接下來幾天，要透過 ArgoCD 這個為了 Kubernetes 與 GitOps 而生的工具，改善這些缺點。

# 什麼是 ArgoCD
![https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcT5JxBwyzDrpsoZJboHIdNCwZMma8GGgQ1uuQ&s](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcT5JxBwyzDrpsoZJboHIdNCwZMma8GGgQ1uuQ&s)

ArgoCD 是一個基於 Pull base GitOps 模型的 Kubernetes 持續交付工具。當 Git Repo 中的 YAML 定義更新時，會將異動自動部署至 Kubernetes，並持續監控 Kubernetes 與 Git Repo 的狀態，並保持狀態同步。

![https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午2.39.43.8s39iv9lry.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午2.39.43.8s39iv9lry.webp)

並提供豐富的開箱即用的功能
- CLI 與 Web UI：有 UI 就給讚
- Webhook：能依據需求進行各種串接
- Notification：輕鬆整合 Slack 等通訊軟體，輕鬆實現告警
- OAuth：能與 Keycloak、Google 等整合，不需額外管理一套使用者帳號

廢話不多說，我們直接來體驗一下 ArgoCD

# 環境準備

## 安裝 ArgoCD
參考[官方安裝文件]，透過 Kustomize 透過 Remote resource 來安裝 ArgoCD
```yaml
# argoCD-demo/infra/argoCD/kustomization.yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: argocd
resources:
- https://raw.githubusercontent.com/argoproj/argo-cd/v2.7.2/manifests/install.yaml
```
部署 ArgoCD 到 Kubernetes 中
```shell
# pwd
# /day19/argoCD-demo <- current folder

# install
kubectl create namespace argocd
kubectl apply -k infra/argoCD
```

## 安裝 ArgoCD CLI
Mac 直接透過 brew 安裝即可，其他平台可參閱[官方文件](https://argo-cd.readthedocs.io/en/stable/cli_installation/#installation)
```shell
# Mac
brew install argocd

# check the install is successful
argocd version
```
## 登入 Web UI
Web UI 的服務也是運行在 Kubernetes 中的 Pod，所以先透過 `kubectl port-forward` 轉發到本地 8080 port，方便 Demo 存取
```shell
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
開啟瀏覽器，連上 `localhost:8080` 應該能看到 ArgoCD 的 Web UI
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午3.58.49.6m3ux6bjew.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午3.58.49.6m3ux6bjew.webp)

透過 `argocd` CLI，取得初始化密碼
```shell
argocd admin initial-password -n argocd

# output
w4sgE3j7Mb97NEzB

 This password must be only used for first time login. We strongly recommend you update the password using `argocd account update-password`.
```
> 📘 改密碼的方式，讀者能自行參閱此篇[官方文件](https://argo-cd.readthedocs.io/en/stable/getting_started/#4-login-using-the-cli)

使用 `admin` 帳號與剛取得的密碼登入
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午4.07.34.4xuhzzwl9n.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午4.07.34.4xuhzzwl9n.webp)

## 透過 ArgoCD 部署服務到 Kubernetes 
直接利用前幾天應用 Kustomize 的 YAML 來部署，先把前幾天使用的 namespace 移除，比較方便看出 ArgoCD 的行為
```shell
# 環境清理
kubectl delete namespace ithome
kubectl create namespace ithome
```

1. 建立 Apps
點選 **NEW APP** 按鈕
![https://argo-cd.readthedocs.io/en/stable/assets/new-app.png](https://argo-cd.readthedocs.io/en/stable/assets/new-app.png)

2. 填寫 Application 基本資料
- Application：`argocd-demo`
- Project：`default`
- Sync Policy：`Automatic` 並勾選 `SELF HEAL`
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午4.23.43.8l01njdag9.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午4.23.43.8l01njdag9.webp)

3. 填寫 Git Repo 資訊   
  使用此 [Github Repo](https://github.com/YihongGao/iThome_30Day_2024) 當作 Manifast Repo，會將 Repo 中的 YAML 部署至 Kubernetes。

- Repository URL：`https://github.com/YihongGao/iThome_30Day_2024`
- Revision：`main`
- Path：`resources/day19/argoCD-demo/apps/overlays/production`
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午4.31.04.8ojnl9h5vw.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午4.31.04.8ojnl9h5vw.webp)

4. 配置要部署到哪個 Kubernetes
- Cluster URL：`https://kubernetes.default.svc`
> 📘 這代表部署到與安裝 ArgoCD 同一個 Kubernetes Cluster
- Namespace：`ithome`
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午4.35.45.1e8ka7xo6f.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午4.35.45.1e8ka7xo6f.webp)

  全部填寫完成後，點選上方 **CREATE** 按鈕
  ![https://argo-cd.readthedocs.io/en/stable/assets/create-app.png](https://argo-cd.readthedocs.io/en/stable/assets/create-app.png)

  到這裡，我們已經在 ArgoCD 上配置了
  - 使用哪個 Git Repo 作為 Kubernetes 期望狀態
  - 要將 YAML 部署到哪個 Kubernetes Cluster 與 namespace
  
  能看到 UI 介面上多了一個叫 argocd-demo 的 App
  ![https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午4.46.42.8hgfpudlgh.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午4.46.42.8hgfpudlgh.webp)

  並且能看到 Status 欄位有兩個狀態機
  第一值是 **App Health** 用來表示該 ArgoCD Application 管理的 Kubernetes 運作是否正常。    

  **App Health** 主要狀態有以下幾個
  - **Healthy**：所有資源運行正常，通過健康檢查，例如 Pod 正常啟動並通過 liveness 和 readiness 檢查。
  - **Progressing**：Application 正在部署或更新中，部分資源尚未部署完成或未達到健康狀態。
  - **Degraded**：部分資源未正常運行，可能有 Pod 啟動失敗或 CrashLoopBackOff 等問題。

  第二個值為 **Sync Status**，主要狀態如下
  - **Synced（已同步）**：Application 的狀態與 Git Repo 完全一致，所有變更已成功應用到 cluster。
  - **OutOfSync（不同步）**：Application 與 Git 中的配置不一致，可能是變更尚未同步到 cluster 或部分資源不符合預期。

  我們來看一下 Kubernetes 是否如 App Status 一樣運作順利。
  
  ```yaml
  kubectl get pod

  # output
  NAME                                READY   STATUS    RESTARTS   AGE
  app-backend-7b8d5c4cd7-ntkhw        1/1     Running   0          2m35s
  app-backend-7b8d5c4cd7-rnxfs        1/1     Running   0          2m24s
  product-backend-5564f9975c-8c4k9    1/1     Running   0          2m9s
  product-backend-5564f9975c-8m2lg    1/1     Running   0          2m30s
  product-schedule-7ccf4f66ff-vfs27   1/1     Running   0          2m27s
  ```
  
  能看到 Git Repo 中的 YAML 都有正確部署到 Kubernetes，接下來我們來嘗試刪除掉任一個 Git Repo 中的 Resource，例如 Deployment
  ```yaml
  kubectl delete deployment app-backend
  ```
  > 📘 若沒有自動建立回來，能檢查一下 argocd-demo 中的 **SYNC POLICY** 是否有 Enable `AUTOMATED` 與 `SELF HEAL`

  能看到該 deployment 會自動被建立回來，這就是 pull base 與 push base GitOps 的最大差異，
  - ArgoCD(pull base) 允許持續維持 Kubernetes 狀態與 Git Repo 中 YAML 的宣告相同。
  - push base 需要依賴頻繁的發佈與嚴謹的工作流程來維持。

# 半自動化的方式
若讀者的公司規範不適合使用全自動化作業時，ArgoCD 也能實現半自動化並提供良好的 Diff 介面來檢視差異。

讀者能透過以下操作體驗看看
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
5. 展開的頁面下方有個 Diff 頁籤，能看到 Git Repo 與 Kubernetes 的當前狀態差異。
  ![https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-18-上午12.49.41.7ljyavd0nn.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-18-上午12.49.41.7ljyavd0nn.webp)
6.當檢視完差異後，若判斷應該要同步到 Kubernetes 時，點選 **SYNC**，就能將 Git Repo 的 YAML 部署到 Kubernetes
  ![https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-18-上午12.51.35.6t72t4yumy.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-18-上午12.51.35.6t72t4yumy.webp)
  

# 小結
今天初步體驗了 ArgoCD 的使用方式 與 ArgoCD 能自動地保持 Git Repo 與 Kubernetes 一致性的功能，這能大幅提高使用 GitOps 模式管理時的信心度與可維護性，避免 Git Repo 與 Kubernetes 環境狀態脫鉤太久後，沒有人有勇氣再次將 Git Repo push 到 Kubernetes 中。

明天會繼續介紹 ArgoCD 的架構與運作原理。

# Refernce
- [ArgoCD 官方文件](https://argo-cd.readthedocs.io/en/stable/)


[官方安裝文件]: https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/#kustomize