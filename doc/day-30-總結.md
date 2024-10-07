# Day-30 總結篇

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

 

# Refernce
- [The journey of a Pod: A guide to the world of Pod Lifecycle](https://www.qikqiak.com/img/posts/pod-workflow.png)
- [Debugging and Monitoring DNS issues in Kubernetes](https://cilium.io/blog/2019/12/18/how-to-debug-dns-issues-in-k8s/)
- [Announcing Hubble - Network, Service & Security Observability for Kubernetes](https://cilium.io/blog/2019/11/19/announcing-hubble/)
- [Dynamically scale your workload with Keda](https://doc.kaas.thalesdigital.io/docs/Features/keda)
- [ArgoCD 官方網站](https://argo-cd.readthedocs.io/en/stable/assets/argocd_architecture.png)
- [Argo Rollouts progressive delivery with Canary deployment](https://jamalshahverdiev.medium.com/argo-rollouts-canary-deployment-5c035ac7a8d4)
- [Feature Toggle Makes Development Faster and Safer @ TECHPULSE 2023](https://speakerdeck.com/line_developers_tw/feature-toggle-makes-development-faster-and-safer-at-techpulse-2023)