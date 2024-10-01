# Day-20 ArgoCD 介紹 - Architecture


# 前言
昨天我們體驗透過 ArgoCD 來實踐 GitOps，利用 Git 來管理/部署 Kubernetes Resource。  

今天讓我們更深入了解 ArgoCD 的架構 與 功能性。

# ArgoCD Architecture
![https://argo-cd.readthedocs.io/en/stable/assets/argocd_architecture.png](https://argo-cd.readthedocs.io/en/stable/assets/argocd_architecture.png)
圖檔來源：[ArgoCD 官方網站](https://argo-cd.readthedocs.io/en/stable/assets/argocd_architecture.png)

在上圖右側區塊中，包含 ArgoCD 最核心的 3 個組件：
- **API Server**：一個 gRPC/REST API 服務，提供 Web UI、CLI 等外部系統的操作入口，主要功能包括：
  - 處理對 Application 的操作請求，如 query、manual sync、rollback
  - 用戶管理，包含 RBAC 和 身份認證
  - 處理 webhook
  - 憑證管理
- **Repository Server**：負責向 Git Repository 獲取 manifest file 的服務：
  - 使用快取提升同步效能 並降低 Git Repository 的負載。
  - 將 Helm chart、Kustomize 等格式轉為 Kubernetes 原生 YAML
- **Application Controller**：類似 Kubernetes Control Plane 的 Controller，主要功能包括：
  - 持續監控 Kubernetes 資源，並將其與 Repository Server 獲取的 manifest files 進行比對，依據 Sync Policy 處理差異。
  - 負責處理 ArgoCD 的 CustomResourceDefinition，如 `applications.argoproj.io`、`appprojects.argoproj.io`

簡單來說，這三個組件各司其職：
- **Repository Server** 負責收集 Git repo 中的 manifest 並轉為原生 YAML，讓 **Application Controller** 與 Git、YAML 管理工具解耦。

- **Application Controller** 則專注在與 Kubernetes API 交互，來同步集群中的資源，確保 Kubernetes 中的應用與 Git 中的定義保持一致。

- 最後由 **API Server** 來補足視圖化、rollback.. 等維護操作等需求，提供完整的功能性給使用者。

## ArgoCD CustomResourceDefinition (CRD)
ArgoCD 主要透過以下兩個 CRD 來管理 GitOps 流程的行為。
- **Application (applications.argoproj.io)**：定義一個 GitOps Application，描述從哪個 Git Repo 中取得 Kubernetes manifests，並如何部署到哪個 Kubernetes Cluster。

- **AppProject (appprojects.argoproj.io)**：管理多個 Application 的範疇及權限，通常能用來避免部署錯環境的問題。

來看看 CRD 的 範例
### Application (applications.argoproj.io)
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd-demo
  namespace: argocd
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: ithome
  source:
    repoURL: https://github.com/YihongGao/iThome_30Day_2024
    path: resources/day19/argoCD-demo/apps/overlays/production
    targetRevision: main
  project: default
  syncPolicy:
    automated:
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
    - ApplyOutOfSyncOnly=true
    - PruneLast=true
```
### 主要欄位與常用選項
- `source`：從哪個 Git repo 取得 manifest file
  - `repoURL`：Git repo 的 URL
  - `path`： manifest file 的路徑
  - `targetRevision`：git 的版本：branch、tag
- `destination`：要部署到哪個 Kubernetes cluster
  - `server`：Kubernetes 的 api server URL
  - `namespace`：部署到哪個 namespace
- `project`：這個 Application 所屬的 AppProject，需滿足 `source` 與 `destination` 的配置需滿足 AppProject 中的條件
- `syncPolicy`：
  - `automated`：配當 Git Repo 發生變化且 Kubernetes 狀態不同步時，自動觸發同步操作，將 Git 配置部署到 Kubernetes。
    - `selfHeal`：開啟此選項後，當 Kubernetes 資源與 Git Repo 不一致時，會自動同步以保持狀態一致。若未啟用，僅當 Git Repo 有變更時才會觸發同步。
      > 📘 單獨使用 `automated`，而未開啟 `selfHeal` 時，只有 Git Repo 有異動時才會觸發部署。
    - `ApplyOutOfSyncOnly`：僅同步狀態不一致的資源，適合管理大量資源的情況，能提升效能並減少對 Kubernetes API Server 的負擔。
    - `PruneLast`：若 Git Repo 中的資源被移除，則同步時會自動從 Kubernetes 中刪除該資源。未啟用時，預設不會自動刪除資源，以防止誤刪重要資源（如 PersistentVolume）。


### AppProject (appprojects.argoproj.io)
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: demo
  namespace: argocd
spec:
  clusterResourceWhitelist:
  - group: '*'
    kind: '*'
  destinations:
  - name: '*'
    namespace: '*'
    server: https://kubernetes.default.svc
  namespaceResourceWhitelist:
  - group: '*'
    kind: '*'
  sourceRepos:
  - https://github.com/YihongGao/iThome_30Day_2024
```
### 主要欄位
- `clusterResourceWhitelist`：定義哪些 Cluster level 的 Resource 能操作，例如 Namespace、PersistentVolume ..等等，若設為 `*` 代表不限制。

- `namespaceResourceWhitelist`：與 `clusterResourceWhitelist` 類似，但管理的是 Namespace level 的 Resource，如 Deployment、Service、Pod。

- `destinations`：允許此 Project 的 Application 能部署到哪個 Kubernetes Cluster 或 namespace。

- `sourceRepos`：允許此 Project 的 Application 能從哪個 Git Repo 取得 manifest。

如果讀者有完成 Day19 的操作，可以使用以下指令檢視昨天透過 UI 建立的 ArgoCD Application 以及預設的 AppProject：
```shell
kubectl get applications.argoproj.io,appprojects.argoproj.io -n argocd 
NAME                                  SYNC STATUS   HEALTH STATUS
application.argoproj.io/argocd-demo   Synced        Healthy

NAME                             AGE
appproject.argoproj.io/default   2d7h
```

您可以透過 UI 介面編輯 Application 或直接操作 CRD 來調整配置。更進一步，您也可以將 ArgoCD 的 CRD 納入 ArgoCD Application 管理，這樣 ArgoCD 本身的配置變更也能透過 GitOps 進行管理與追蹤。

# 小結
今天介紹了 ArgoCD 的架構 與 CRD，明天會介紹其他 ArgoCD 的進階/管理功能應用。

# Refernce
- [ArgoCD 官方文件](https://argo-cd.readthedocs.io/en/stable/)
