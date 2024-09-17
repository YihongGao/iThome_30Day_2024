
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
- 
# 小結

# Refernce


[官方安裝文件]: https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/#kustomize