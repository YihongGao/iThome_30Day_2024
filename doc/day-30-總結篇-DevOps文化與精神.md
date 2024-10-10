# Day-30 總結篇 - DevOps 文化/精神

終於來到了最後一天，來總結一下這一個月來所涵蓋的主題。

# 回顧主題
## Kubernetes Architecture
- **深入介紹 Kubernetes 架構**：理解 API Server、etcd、Controller Manager、kubelet、kube-proxy 的核心組件。
- **了解 Pod 在 Kubernetes 的旅程**：如何通過 認證、授權、分配調度 最終與 CRI、CNI、CSI 協作來運行 Pod。
![https://miro.medium.com/v2/format:webp/1*WDJmiyarVfcsDp6X1-lLFQ.png](https://miro.medium.com/v2/format:webp/1*WDJmiyarVfcsDp6X1-lLFQ.png)
圖片來源：[The journey of a Pod: A guide to the world of Pod Lifecycle](https://www.qikqiak.com/img/posts/pod-workflow.png)
-  **流量分配原理**：理解如何透過 DNS 與 kube-proxy 實現 Kubernetes Service。
![https://cilium.io/static/7720169a677cd13bbad2b9c431d560d8/1ab28/ogimage.webp](https://cilium.io/static/7720169a677cd13bbad2b9c431d560d8/1ab28/ogimage.webp)
圖檔來源 : [Debugging and Monitoring DNS issues in Kubernetes](https://cilium.io/blog/2019/12/18/how-to-debug-dns-issues-in-k8s/)
- **Pod 的分配策略**：透過 NodeSelector、Node Affinity、Taints 和 Tolerations 控制 Pod 分配，優化 Pod 的分佈，確保效能與高可用性。
- **Kubernetes 的網路安全**：透過 Network Policy 控制 Pod 間的網路流量及外部訪問，確保僅授權的流量能夠通過，提升安全性，並透過 Cilium 與 Hubble 觀測流量。
![https://cilium.io/static/2e1ecf6fa94e4a3450155a4660881927/83a93/servicemap.webp](https://cilium.io/static/2e1ecf6fa94e4a3450155a4660881927/83a93/servicemap.webp)
圖檔來源：[Announcing Hubble - Network, Service & Security Observability for Kubernetes](https://cilium.io/blog/2019/11/19/announcing-hubble/)

## Kubernetes Event-driven Autoscaling
- **更敏銳、多元的擴/縮容策略**：允許使用 **Prometheus、Message Queue**..等數據或狀態作為 擴/縮容依據。
- **良好的資源利用**：允許 Pod 副本數降為 0，讓 Cluster 資源利用率更佳。
![https://doc.kaas.thalesdigital.io/assets/images/keda_context-56c4cbf1f0f045877ea8d93878839cbc.png](https://doc.kaas.thalesdigital.io/assets/images/keda_context-56c4cbf1f0f045877ea8d93878839cbc.png)
圖檔來源：[Dynamically scale your workload with Keda](https://doc.kaas.thalesdigital.io/docs/Features/keda)

## GitOps
- **學習 Kustomize**：透過模組化管理 Kubernetes YAML，更容易管理多環境配置。
- **運用 Argo CD 實踐 Pull-based GitOps**：實現配置與環境強一致性 與 避免 kube-api-sever 與 Access token 曝露。
![https://argo-cd.readthedocs.io/en/stable/assets/argocd_architecture.png](https://argo-cd.readthedocs.io/en/stable/assets/argocd_architecture.png)
圖檔來源：[ArgoCD 官方網站](https://argo-cd.readthedocs.io/en/stable/assets/argocd_architecture.png)

## 部署策略
- **了解多種部署策略**：透過 Recreate、Rolling Update、Blue/Green、Canary 等部署方式，減少軟體發佈的風險。
- **如何實踐部署策略於 Kubernetes**：使用 Ingress NGINX Controller、Argo Rollouts 等組件輕鬆的在 Kubernetes 作為實現漸進式部署（Progressive Delivery）的解決方案。
![https://miro.medium.com/v2/resize:fit:1400/format:webp/1*HlRY7yeADNQe54zfoWSp2w.png](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*HlRY7yeADNQe54zfoWSp2w.png)
圖檔來自：[Argo Rollouts progressive delivery with Canary deployment](https://jamalshahverdiev.medium.com/argo-rollouts-canary-deployment-5c035ac7a8d4)

- **Feature Toggle**：透過 Flipt 實現 Feature Toggle 使用更低成本、更小維度的方式實踐 漸進式部署（Progressive Delivery）或 A/B Testing。
![https://github.com/YihongGao/picx-images-hosting/raw/master/20241002/截圖-2024-10-02-下午9.32.25.73tx7ivvde.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20241002/截圖-2024-10-02-下午9.32.25.73tx7ivvde.webp)
圖檔來自 [Feature Toggle Makes Development Faster and Safer @ TECHPULSE 2023](https://speakerdeck.com/line_developers_tw/feature-toggle-makes-development-faster-and-safer-at-techpulse-2023)

## DevSecOps
- **安全性測試左移**：將弱點掃描融入 CI/CD 流程，取代傳統高延遲的季/年掃描計畫，避免弱點進入生產環境。
- **自動化整合**：將掃描作業自動化，避免增加人力成本 與 產生人為失誤。
 ![https://github.com/YihongGao/picx-images-hosting/raw/master/20241007/截圖-2024-10-07-下午7.48.55.4jo31xe2td.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20241007/截圖-2024-10-07-下午7.48.55.4jo31xe2td.webp)

# DevOps 文化/精神
在這個月的學習過程中，讀者應該已經注意到，我們討論的主題不僅涉及到開發者（Developer）的技能，還包括許多維運人員（Operator）日常關注的知識與工具。這正是 [DevOps](https://aws.amazon.com/tw/devops/what-is-devops/) 文化的核心精神——打破開發與運維之間的隔閡，讓團隊共同負責應用服務的 **可靠性（Reliability）**、**擴展性（Scalability）**、**可維護性（Maintainability）** 和 **安全性（Security）**。

DevOps 強調跨團隊協作，通過自動化工具和流程統一管理應用的開發、部署和運維，實現更快速、可靠的軟體交付。透過這種文化，開發者和運維人員能夠共同掌握應用的全生命周期，從開發測試到生產環境的管理，讓軟體應用有更好的品質。

# 與 Kubernetes 相伴的未來

## AI
在這個 AI 崛起的時代，除了 Copilot 這類 AI 協作軟體開發工具，也許能嘗試讓 AI 技術來協助我們 Kubernetes 的應用，如 [K8sGPT](https://k8sgpt.ai/) 等工具，讓 生成式 AI 協助我們進行錯誤診斷 與 問題排查，來更輕鬆的使用/管理 Kubernetes。

## Service Mash
隨著微服務架構的普及，Kubernetes Cluster 中通常會運行大量微服務，甚至有多 Cluster 之間需要互相溝通，這使得開發/管理越來越複雜。透過 [Istio](https://istio.io/) 這類 **Service Mash** 的解決方案，能避免對應用程序做侵入式的改動，透過 eBPF 或 Sidercar 來管理/控制網路流量。    

### 使用案例：
- [整合 OpenTelemetry](https://istio.io/latest/docs/tasks/observability/distributed-tracing/opentelemetry/)
- [實現 mTLS](https://istio.io/latest/docs/tasks/security/authentication/mtls-migration/)
- [實現 熔斷（Circuit Breaking）](https://istio.io/latest/docs/tasks/traffic-management/circuit-breaking/)


## Serverless
Kubernetes 也能透過 [Knative](https://knative.dev/docs/) 這樣的工具實現 [Serverless](https://aws.amazon.com/cn/blogs/china/iaas-faas-serverless/)，提供統一的部署方式，讓開發者專注在商業邏輯，減少對底層的知識門檻。

**Serverless** 基於 Http/Event 的擴展能力，也能應用在購物網站這類流量波鋒特別大的服務，搭配具有快速啟動的應用程序（如 [Golang](https://go.dev/) 或 [native-image](https://www.graalvm.org/latest/reference-manual/native-image/)），即能降低冷啟動的延遲問題。

# 結語
感謝各位的閱讀，以上就是今年鐵人賽的分享，希望能讓大家理解 Kubernetes 的運作原理 與 更多應用方式，透過這些知識讓 Kubernetes 與 應用程序 產生更深度的協作，提供夠穩定、可靠的服務。

感謝各位的閱讀，透過今年的鐵人賽分享，希望有讓大家對 Kubernetes 的運作原理有了更深入的理解，並學到了更多應用的實踐方式。希望這些知識能幫助大家在工作中與 Kubernetes 建立更緊密的協作關係，打造出穩定、可靠且高效的應用服務。




# Refernce
- [The journey of a Pod: A guide to the world of Pod Lifecycle](https://www.qikqiak.com/img/posts/pod-workflow.png)
- [Debugging and Monitoring DNS issues in Kubernetes](https://cilium.io/blog/2019/12/18/how-to-debug-dns-issues-in-k8s/)
- [Announcing Hubble - Network, Service & Security Observability for Kubernetes](https://cilium.io/blog/2019/11/19/announcing-hubble/)
- [Dynamically scale your workload with Keda](https://doc.kaas.thalesdigital.io/docs/Features/keda)
- [ArgoCD 官方網站](https://argo-cd.readthedocs.io/en/stable/assets/argocd_architecture.png)
- [Argo Rollouts progressive delivery with Canary deployment](https://jamalshahverdiev.medium.com/argo-rollouts-canary-deployment-5c035ac7a8d4)
- [Feature Toggle Makes Development Faster and Safer @ TECHPULSE 2023](https://speakerdeck.com/line_developers_tw/feature-toggle-makes-development-faster-and-safer-at-techpulse-2023)

- [AWS/ What is DevOps](https://aws.amazon.com/tw/devops/what-is-devops/)

