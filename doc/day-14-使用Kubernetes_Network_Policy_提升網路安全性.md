
# Day-14 使用 Kubernetes NetworkPolicy 提升網路安全性

# 前言
在傳統的 VM 或實體伺服器架構中，我們經常會將伺服器分層，並對不同層的服務進行網路隔離，僅允許必要的網路連線，從而降低服務被攻擊時的風險。
> 📘 常見的案例如: [three-tier architecture]

預設情況下，Kubernetes 中的每個 Pod 都能互相通訊，這雖然方便我們快速構建服務，但也帶來了安全隱患。假設其中一個 Pod 被攻擊者入侵，攻擊者可能通過 [橫向移動 (lateral movement)] 進一步攻擊其他 Pod 或內部服務（如資料庫）。

今天要介紹的 [NetworkPolicy] 是一種用來管理 Pod 網路流量的資源，能有效限制流量的來源與去向，從而阻止攻擊者橫向擴展，並降低安全事件的影響範圍。

# NetworkPolicy
NetworkPolicy 是用來控制 Pod 間以及 Pod 與外部網路流量的策略工具。透過定義規則，NetworkPolicy 可以指定哪些 Pod 可以彼此通訊，以及哪些 Pod 可以與外部資源互動。

簡單來說，它可以視為 Kubernetes 內部的防火牆。
![https://miro.medium.com/v2/resize:fit:1400/format:webp/1*HtExF_QoSjK7MsL4-218PA.png](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*HtExF_QoSjK7MsL4-218PA.png)
圖檔來源 : [Deep-dive: Kubernetes NetworkPolicy in GKE]

NetworkPolicy 將流量分為
- **Ingress**：即入站流量。任何向 Pod 發送的流量都會通過 Ingress 規則進行判斷，決定是否允許或拒絕該流量。

- **Egress**：即出站流量。任何從 Pod 發出的流量都會通過 Egress 規則進行判斷，決定是否允許或拒絕。 
  > 📘 例如，當 Pod 連向其他 Pod、外部服務或資料庫時，這些流量屬於 Egress。

## NetworkPolicy 範例
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```
主要欄位介紹
- `spec`
  - `podSelector`：使用 Label selector 指定這個 NetworkPolicy 作用於哪些 Pod。這樣可以控制 NetworkPolicy 只影響具有特定標籤的 Pod。
  
  - `policyTypes`：定義 NetworkPolicy 管理的流量類型。可選擇 `Ingress`（入站流量）、`Egress`（出站流量），或同時管理兩者。

  - `ingress.from`：定義允許哪些入站流量，主要子欄位包括：
    - ``ipBlock``：透過 `cidr` 指定允許哪些 IP 範圍進入 Pod。可以使用 `except` 排除某些 IP 區段，以更精確控制流量。
    
    - `namespaceSelector`：透過 Label selector 允許來自具有指定 Label 的 namespace 中所有 Pod 的流量進入。
    
    - `podSelector`：允許來自具有指定 Label 的 Namespace 中所有 Pod 的流量。
    - `ports`：允許來自具有指定 Label 的 Pod 流量，確保只有特定標籤的 Pod 可以與目標 Pod 通訊。

  - `egress.to`：定義指定 Pod 能夠發送流量的目的地。
    > 📘 `egress.to` 的結構與 `ingress.from` 相同，使用`ipBlock`、`namespaceSelector`、`podSelector`、`port` 來控制出站流量（Egress）的目的地和通訊協定。

## 使用案例
透過上面的範例，我們了解到可以使用 Label selector 將 NetworkPolicy 套用到特定的 Pod 上。我們可以借鑒 [three-tier architecture] 將服務大致分為不同 tier，並搭配 NetworkPolicy 來降低資料洩漏的風險。
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240907/截圖-2024-09-07-下午10.36.16.70aaa5clyg.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240907/截圖-2024-09-07-下午10.36.16.70aaa5clyg.webp)

- `Frontend / Gateway tier`: 負責接受來自 Cluster 外部的流量，通常處理認證和授權，並將合法流量轉發至內部服務。常見的服務如 API Gateway、Nginx 等。
- `Backend tier`: 處理商業邏輯的服務，通常是 API 服務，如 Tomcat 或 Django。
- `Data tier`: 負責資料存儲的服務，如資料庫。

### NetworkPolicy 概念轉換
將上述概念應用到 NetworkPolicy，可大致定義以下規則：
- **Default NetworkPolicy**：不允許任何入站/出站流量。
- **Frontend / Gateway tier NetworkPolicy**
  - ****ingress****：允許來自外部的全部或特定流量進入該 tier 的 Pod。
  - ****egress**** ：僅允許將流量發送至 Backend tier。
- **Backend tier NetworkPolicy**：
  - ****ingress****：僅允許來自 Frontend / Gateway tier 的流量進入。
  - ****egress**** ：僅允許將流量發送至 Data tier 或其他 Backend 服務。
- **Data tier NetworkPolicy**：
  - ****ingress****：僅允許來自 Backend tier 的流量進入。
  - ****egress**** ：不允許任何出站流量，以保護數據安全。

讓我們來一步一步進行配置

> 代碼能參閱 [GitHub](https://github.com/YihongGao/iThome_30Day_2024/tree/main/resources/day14)

1. Default NetworkPolicy
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: default-deny-all
    spec:
      podSelector: {}
      policyTypes:
      - Ingress
      - Egress
    ```
- `policyTypes`: 指定了 `Ingress` 與 `Egress` 且未設定 `ingress` 與 `egress` 區塊代表不允許任何入站與出站流量
- `podSelector`: `{}` 代表選擇所有 Pod。

    2. Frontend / Gateway tier NetworkPolicy
    ```yaml 
    # 套用到 tier: frontend 的 NetworkPolicy
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: frontend-tier-policy
    spec:
      podSelector:
        matchLabels:
          tier: frontend
      policyTypes:
      - Ingress
      - Egress
      ingress:
      - {}
      egress:
      - to:
        - podSelector:
            matchLabels:
              tier: backend
    ```
    ```yaml 
    # 套用到 tier: gateway 的 NetworkPolicy
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: gateway-tier-policy
    spec:
      podSelector:
        matchLabels:
          tier: gateway
      policyTypes:
      - Ingress
      - Egress
      ingress:
      - {}
      egress:
      - to:
        - podSelector:
            matchLabels:
              tier: backend
    ```
    兩個 NetworkPolicy 分別套用到 `tier=frontend` 或 `gateway` 的 Pod，允許任何流量進入，但只能將流量送往有 `tier=backend` 的 Pod，不可直接存取資料庫。

3. Backend tier NetworkPolicy

    ```yaml 
    # 套用到 tier: backend 的 NetworkPolicy
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: backend-tier-policy
    spec:
      podSelector:
        matchLabels:
          tier: backend
      policyTypes:
      - Ingress
      - Egress
      ingress:
      - from:
        - podSelector:
            matchLabels:
              tier: frontend
        - podSelector:
            matchLabels:
              tier: gateway
      egress:
      - to:
        - podSelector:
            matchLabels:
              tier: data
      - to:
        - podSelector:
            matchLabels:
              tier: backend
    ```
    允許 `tier=frontend` 或 `gateway` 的 Pod 傳入流量，避免 backend 服務被外部直接攻擊，且只能將流量送往同層的 Pod (`tier=backend`) 或 資料層的 Pod (`tier=data`)  的 Pod。

4. Data tier NetworkPolicy:

    ```yaml 
    # 套用到 tier: backend 的 NetworkPolicy
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: data-tier-policy
    spec:
      podSelector:
        matchLabels:
          tier: data
      policyTypes:
      - Ingress
      - Egress
      ingress:
      - from:
        - podSelector:
            matchLabels:
              tier: backend
    ```
僅允許 `tier=backend` 的 Pod 傳入流量，但不允許任何出站流量，減少資料被直接從內部搬出的風險。

# 注意事項
並不是每個 Kubernetes 都支援 NetworkPolicy，需要該 Kubernetes 使用的 CNI，否則配置的 NetworkPolicy，不會發揮作用也不會給出提示錯誤訊息，參考[官方文件](
https://kubernetes.io/docs/concepts/services-networking/network-policies/#prerequisites)
> 📘 kind 預設使用的 CNI 不支援 NetworkPolicy

# 小結
今天我們介紹了 NetworkPolicy 這個 Kubernetes 管理網路流量的資源，並參考了傳統的三層架構來提升服務的安全性。讀者可以根據需求進一步自訂 NetworkPolicy，例如：
- 每個 Pod 只允許存取其依賴的 Pod 或外部服務，而非同層級的所有 Pod。

明天的文章會帶讀者安裝 [Cilium] 作為 CNI，並透過 [Hubble] 來觀察和監控網路流量。


# Refernce
- [Kubernetes 官方文件](https://kubernetes.io/zh-cn/docs/concepts/services-networking/network-policies/#networkpolicy-resource)
- [Deep-dive: Kubernetes NetworkPolicy in GKE]

[Deep-dive: Kubernetes NetworkPolicy in GKE]: https://medium.com/google-cloud/deep-dive-kubernetes-network-policy-in-gke-e9842ec6b1be

[橫向移動 (lateral movement)]: https://www.wiz.io/blog/lateral-movement-risks-in-the-cloud-and-how-to-prevent-them-part-2-from-k8s-clust
[three-tier architecture]: https://www.ibm.com/topics/three-tier-architecture

[cilium]: https://docs.cilium.io/en/stable/overview/intro/

[hubble]: https://docs.cilium.io/en/stable/overview/intro/#what-is-hubble

[NetworkPolicy]: https://kubernetes.io/docs/concepts/services-networking/network-policies/