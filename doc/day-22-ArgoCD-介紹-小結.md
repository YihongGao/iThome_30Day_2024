
# Day-22 ArgoCD 介紹 - 小結

# 前言
透過這幾天了解 ArgoCD 的架構與功能後，我們來看看是否改善了 Push-base 的 GitOps 的缺點。 
  
1. 安全性風險：Push 模式需要暴露 cluster 的 API server，增加公開網絡中的風險。
2. 難以擴展：隨著集群數量增加，Push 模式在多個 cluster 的管理上變得困難，尤其是跨網域時。
3. 一致性問題：若有人為操作 Kubernetes，可能導致 cluster 與 Git Repo 不同步。

**Push-Based GitOps flow**
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午2.39.33.26lfru5jsv.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午2.39.33.26lfru5jsv.webp)

## 安全性風險
在使用 Push-based 模式時，CI/CD pipeline 的伺服器必須主動連線到 Kubernetes API server，並攜帶足夠權限的 access token。這種操作方式帶來了以下幾個主要風險：
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240921/截圖-2024-09-21-下午4.45.26.13lqmsiv52.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240921/截圖-2024-09-21-下午4.45.26.13lqmsiv52.webp) 

這個操作方式，會產生以下風險
- **增加攻擊位面**：當有更多伺服器具備存取 Kubernetes Cluster 的能力時，攻擊者潛在的入侵路徑也增加了。如果 CI/CD server 遭到攻擊，整個 Cluster 可能面臨更大的風險。

- **Access Token 竊取風險**：CI/CD Pipeline 通常需要較高權限的 access token。這些 token 若儲存在伺服器中且被竊取，攻擊者可能會使用它們來掌控 Kubernetes Cluster，導致服務中斷或被劫持。

透過 ArgoCD 使用 Pull-based 模式，由 Kubernetes Cluster 主動從外部 Git Repo 拉取變更，避免外部伺服器直接連線到 Kubernetes API server。
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240921/截圖-2024-09-21-下午5.02.07.77dip96w3w.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240921/截圖-2024-09-21-下午5.02.07.77dip96w3w.webp)

這個方式類似於軟體開發中的 **依賴反轉原則**，將原本 CI/CD Server 依賴 Kubernetes API server 的模式，轉變為 Kubernetes Cluster 依賴外部的 Git Repo 來同步變更。這樣做有以下優點：

- **降低安全風險**：CI/CD Server 無需具備存取 Kubernetes 的權限，不必持有 Kubernetes API 的 access token，極大地減少了攻擊面。
- **權限隔離**：CI/CD Server 不需要了解 Kubernetes 的任何內部資訊，權限分離更加徹底。
 
## 難以擴展
當要透過 Push-based 模式要管理更多 Kubernetes Cluster 時，勢必要開通更多網路連線與配置更多把 Access token，而且通常是跨環境(開發、生產)的情境，在許多公司的安全政策中是不允許的。

透過 ArgoCD 能減少網路存取 並 更容易進行權限隔離，讓管理多 Cluster 不在綁手綁腳，例如

### Multiple ArgoCD 
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240921/截圖-2024-09-21-下午5.24.39.67xfc3x1cy.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240921/截圖-2024-09-21-下午5.24.39.67xfc3x1cy.webp)
讓每個環境有自己的 ArgoCD，網路連線需求就只要每個 Cluster 都能存取該 Git Repo(Manifest Repo) 即可，再搭配前幾天介紹的 Kustomize overlays 就能兼顧多環境的 YAML 可維護性。

### 中心輻射模型 (Hub-Spoke model)
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240921/截圖-2024-09-21-下午5.44.34.51e43ixuhf.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240921/截圖-2024-09-21-下午5.44.34.51e43ixuhf.webp)
透過 Hub Cluster，使用單一個 ArgoCD 管理多個 Kubernetes Cluster，能減少管理多個 ArgoCD 的維運成本 與 更容易進行權限控管與配置。

## 一致性問題
在 Push-based 模式 中當有人透過 kubectl 直接操作 Kubernetes 時，會導致配置與環境不相同，且不易防範，容易導致維運人員對 Git 中的配置心存疑慮。
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240921/截圖-2024-09-21-下午5.57.28.9dcxb324tk.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240921/截圖-2024-09-21-下午5.57.28.9dcxb324tk.webp)

透過 ArgoCD 自我修復功能（Self-Heal），當直接操作 ArgoCD 管理的 Resource 時，會自動將其恢復與 GitOps 配置相符，達成強一致性，讓維運人員更容易管理與對配置檔更有信心。

# 小結
透過 ArgoCD 引入 Pull-based 模式，有效解決 Push-based GitOps 在安全性、擴展性與一致性上的挑戰。這不僅降低了 Kubernetes API server 暴露於外部的風險，還通過 Hub-Spoke 模型與 Multiple ArgoCD 架構提升了多叢集管理的靈活性與可擴展性。此外，ArgoCD 的自我修復功能（Self-Heal）也能確保集群配置與 Git Repo 一致，避免手動操作帶來的配置差異，進一步提升了系統穩定性與維運信心。

總體而言，ArgoCD 的引入不僅簡化了運維工作，還提升了整體系統的安全性與可靠性，為現代化 Kubernetes 環境下的持續交付提供了強大的解決方案。

# Refernce
- [ArgoCD 官方文件](https://argo-cd.readthedocs.io/en/stable/)


[官方安裝文件]: https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/#kustomize