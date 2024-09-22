
# Day-20 ArgoCD 介紹 - Architecture


# 前言
昨天我們體驗透過 ArgoCD 來實踐 GitOps，利用 Git 來管理/部署 Kubernetes Resource。  

今天讓我們更深入了解 ArgoCD 的架構 與 功能性。

# ArgoCD Architecture
![https://argo-cd.readthedocs.io/en/stable/assets/argocd_architecture.png](https://argo-cd.readthedocs.io/en/stable/assets/argocd_architecture.png)
圖檔來源：[ArgoCD 官方網站](https://argo-cd.readthedocs.io/en/stable/assets/argocd_architecture.png)

於上圖中有右邊區塊中的 3 個組件，是 Argo CD 最核心的部分
- **API Server**：一個 gRPC/REST 的 API 服務，提供各式 API 給 Web UI、CLI 等外部使用。功能包含
  - 處理對 application 的操作請求，如 query、manual sync、rollback
  - 用戶管理，包含 RBAC、身份認證
  - 處理 webhook
  - 憑證管理
- **Repository Server**：負責向 Git Repository 溝通獲取 manifest file 的服務
  - 使用快取提升同步效能 並降低 Git Repository 的負載。
  - 將 Helm chart、Kustomize 等格式轉為 Kubernetes 原生 YAML
- **Application Controller**：類似 Kubernetes Controll plane 中的 Controller，
  - 持續監控 Kubernetes 中的 Resource 並與 Repository Server pull 到的 manifest file 進行比對，當發現差異時，依據 Sync Policy 進行處理。
  - 負責處理 ArgoCD 的 CustomResourceDefinition，如 `applications.argoproj.io`、`appprojects.argoproj.io`

簡單來說，這三個組件各司其職 **Repository Server** 負責收集 Git repo 中的 manifest 並轉為原生 YAML，讓 **Application Controller** 與 Git、YAML 管理工具解耦。

**Application Controller** 則專注在與 Kubernetes API 交互，來同步集群中的資源，確保 Kubernetes 中的應用與 Git 中的定義保持一致。

最後由 **API Server** 來補足視圖化、rollback..等維護操作等需求，提供完整的功能性給使用者。

## ArgoCD CustomResourceDefinition (CRD)
ArgoCD 主要透過以下兩個 CRD 來管理 GitOps 流程的行為。
- **Application (applications.argoproj.io)**：定義一個 GitOps Application，描述從哪個 Git Repo 中取得 Kubernetes manifests，並如何部署到哪個 Kubernetes Cluster。

- **AppProject (appprojects.argoproj.io)**：管理多個 Application 程式的範疇及權限，通常能用來避免部署錯環境的問題。

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
介紹幾個主要欄位 與 常用的選項
- `source`：從哪個 Git repo 取得 manifest file
  - `repoURL`：repo URL
  - `path`： manifest file 的路徑
  - `targetRevision`：git 的版本：branch、tag
- `destination`：要部署到哪個 Kubernetes cluster
  - `server`：Kubernetes 的 api server URL
  - `namespace`：部署到哪個 namespace
- `project`：這 Application 屬於哪個 AppProject，需滿足 `source` 與 `destination` 的配置需滿足 AppProject 中的條件
- `syncPolicy`：
  - `automated`：配置此屬性代表當偵測到 Git Repo 發生變化且與 Kubernetes 當前狀態不相同時，要觸發部署行為將 Git Repo 的配置部署到 Kubernetes 中，以保持狀態同步。
    - `selfHeal`：預設配置下，若有人直接操作 Kubernetes Resource 導致狀態不同步，ArgoCD 不會觸發部署操作來保持同步。當開啟此選項時，當偵測到 Kubernetes Resource 與 Git Repo 期望狀態不一致時，會觸發部署操作，以保持兩端一致。
      > 📘 單獨使用 `automated`，而未開啟 `selfHeal` 時，只有 Git Repo 有異動時才會觸發部署。
    - `ApplyOutOfSyncOnly`：部署操作是否只部署有差異的 Resource。當 ArgoCD 管理的 Resource 眾多時，能提高效能減少 kube-api-server 的負擔。
    - `PruneLast`：預設情況下，若移除了 Git Repo 中的 Resource，再 ArgoCD 進行部署操作時，不會自動將該 Resource 從 Kubernetes 中刪除，避免誤操作造成不可回復的問題(如刪 pv)，開啟此選項才會自動刪除。

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
介紹主要欄位
- `clusterResourceWhitelist`：定義哪些 Cluster level 的 Resource 能操作，例如 Namespace、PersistentVolume ..等等，若設為 `*` 代表不限制。

- `namespaceResourceWhitelist`：與 `clusterResourceWhitelist` 類似，但管理的是 Namespace level 的 Resource，如 Deployment、Service、Pod。

- `destinations`：允許此 Project 的 Application 能部署到哪個 Kubernetes Cluster 或 namespace。

- `sourceRepos`：允許此 Project 的 Application 能從哪個 Git Repo 取得 manifest。

若讀者有進行 Day19 的演練操作，能使用以下指令看到昨天透過 UI 建立的 ArgoCD application 與 預設的 AppProject
```shell
kubectl get applications.argoproj.io,appprojects.argoproj.io -n argocd 
NAME                                  SYNC STATUS   HEALTH STATUS
application.argoproj.io/argocd-demo   Synced        Healthy

NAME                             AGE
appproject.argoproj.io/default   2d7h
```

使用時能透過 UI 編輯 或 操作 CRD 來調整配置，甚至將 ArgoCD CRD 也納入 ArgoCD Application 管理的資源，讓 ArgoCD manifest 的異動也能透過 GitOps 進行管理與追蹤。

# 小結
今天介紹了 ArgoCD 的架構 與 CRD，明天會介紹其他 ArgoCD 的進階/管理功能應用。

# Refernce
- [ArgoCD 官方文件](https://argo-cd.readthedocs.io/en/stable/)
