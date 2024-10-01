# Day-22 ArgoCD 介紹 - 小結

# 前言
透過這幾天了解 ArgoCD 的架構與功能後，我們來看看是否改善了 Push-base 的 GitOps 的一些缺點。 
  
1. 安全性風險：Push 模式需要暴露 cluster 的 API server，增加公開網絡中的風險。
2. 難以擴展：隨著集群數量增加，Push 模式在多個 cluster 的管理上變得困難，尤其是跨網域時。
3. 一致性問題：若有人為操作 Kubernetes，可能導致 cluster 與 Git Repo 不同步。

**Push-Based GitOps flow**
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午2.39.33.26lfru5jsv.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240917/截圖-2024-09-17-下午2.39.33.26lfru5jsv.webp)

## 安全性風險
在 Push-based 模式中，CI/CD Pipeline Server 必須主動連接到 Kubernetes API server，並使用高權限的 Access Token。這種方式帶來了以下風險：
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240921/截圖-2024-09-21-下午4.45.26.13lqmsiv52.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240921/截圖-2024-09-21-下午4.45.26.13lqmsiv52.webp) 

- **增加攻擊位面**：更多伺服器擁有存取 Kubernetes Cluster 的能力，會增加潛在的入侵路徑。如果 CI/CD 伺服器遭到攻擊，整個 Cluster 可能面臨風險。

- **Access Token 竊取風險**：CI/CD Pipeline 需要持有高權限的 Access Token。如果這些 Token 被竊取，攻擊者可能會獲得對 Kubernetes Cluster 的控制權，造成服務中斷或數據劫持。

透過 ArgoCD 的 Pull-based 模式，Kubernetes Cluster 主動從外部 Git Repo 拉取變更，避免外部伺服器直接連接到 Kubernetes API server。
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240921/截圖-2024-09-21-下午5.02.07.77dip96w3w.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240921/截圖-2024-09-21-下午5.02.07.77dip96w3w.webp)

這種方式類似於軟體設計中的 **依賴反轉原則**，將原本由 CI/CD Server 依賴 Kubernetes API server 的模式，轉變為 Kubernetes Cluster 依賴 Git Repo 進行同步變更。這樣的做法帶來以下優點：

- **降低安全風險**：CI/CD Server 無需持有 Kubernetes API 的 Access Token，顯著減少了攻擊面。
- **權限隔離**：CI/CD Server 無需知道 Kubernetes 的內部信息，權限分離更加徹底。
 
## 難以擴展
在 Push-based 模式下，管理多個 Kubernetes Cluster 時，必須開通更多的網路連線，並配置多個 Access Token。尤其在跨環境（如開發與生產）的情況下，這種方式常常違反許多公司的安全政策。

使用 ArgoCD 可以減少網絡存取需求，並輕鬆實現權限隔離，讓多 Cluster 的管理變得更加靈活。以下是兩種常見的架構模式：

### Multiple ArgoCD 模式
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240921/截圖-2024-09-21-下午5.24.39.67xfc3x1cy.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240921/截圖-2024-09-21-下午5.24.39.67xfc3x1cy.webp)
在此模式下，每個環境（如開發和生產）有自己獨立的 ArgoCD，這樣每個 Cluster 只需要能存取 Git Repo（即 Manifest Repo）即可。結合 Kustomize 的 overlays 配置，我們可以輕鬆維護多個環境的 YAML 文件。

### 中心輻射模型 (Hub-Spoke model)
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240921/截圖-2024-09-21-下午5.44.34.51e43ixuhf.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240921/截圖-2024-09-21-下午5.44.34.51e43ixuhf.webp)
在這種中心輻射模型中，使用單一的 ArgoCD（部署在 Hub Cluster 中）來管理多個 Kubernetes Cluster。這樣可以減少維運多個 ArgoCD 的成本，同時更容易進行權限管理和配置。

## 一致性問題
在 Push-based 模式中，如果有人通過 kubectl 直接操作 Kubernetes，會導致環境與 Git 中的配置不同步，這很難被防範，也會讓維運人員對 Git 配置失去信心。
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240921/截圖-2024-09-21-下午5.57.28.9dcxb324tk.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240921/截圖-2024-09-21-下午5.57.28.9dcxb324tk.webp)

透過 ArgoCD 的自我修復功能（Self-Heal），當有人直接操作由 ArgoCD 管理的資源時，系統會自動將其恢復到與 Git 中一致的狀態，實現強一致性，並增強維運人員對配置的信心。

# 小結
透過 ArgoCD 引入 Pull-based 模式有效解決 Push-based GitOps 在安全性、擴展性與一致性上的挑戰。它不僅降低了 Kubernetes API server 暴露於外部的風險，還通過 Hub-Spoke 模型與 Multiple ArgoCD 架構提升了多叢集管理的靈活性與可擴展性。此外，ArgoCD 的自我修復功能（Self-Heal）也能確保集群配置與 Git Repo 一致，避免手動操作帶來的配置差異，進一步提升了系統穩定性與維運信心。

總體而言，ArgoCD 不僅簡化了運維工作，還提升了系統的安全性與可靠性，成為現代化 Kubernetes 環境中強大的持續交付解決方案。

# Refernce
- [ArgoCD 官方文件](https://argo-cd.readthedocs.io/en/stable/)
