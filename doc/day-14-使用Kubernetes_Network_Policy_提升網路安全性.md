
# Day-14 使用 Kubernetes NetworkPolicy 提升網路安全性

# 前言
以往透過 VM 或 實體機組成的伺服器架構，時常會將伺服器進行分層，將不同層的服務進行網路隔離，只允許必要的網路連線，降低服務被侵入時的風險。
> 📘 常見的案例如: [three-tier architecture]

預設 Kubernetes 中的每個 Pod 都能互相進行網路通訊，讓我們能快速簡單的建構服務，但這也隱含了一些安全問題。假設有一個 Pod 被入侵時，攻擊者能再 Cluster 中進行 [橫向移動 (lateral movement)] 輕易攻擊其他 Pod 或 內部服務(如資料庫)。

今天要介紹的 [NetworkPolicy] 能對 Pod 的網路流量進行管理，來讓攻擊者更難得手 與 降低安全事件的影響範圍。

# NetworkPolicy
NetworkPolicy 主要用來控制 Pod 間以及 Pod 與外部網路的流量。透過定義規則，NetworkPolicy 可以指定哪些 Pod 可以互相通訊，以及哪些 Pod 可以與外部資源交互。

簡單來說，能當作 Kubernetes 內部的防火牆。
![https://miro.medium.com/v2/resize:fit:1400/format:webp/1*HtExF_QoSjK7MsL4-218PA.png](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*HtExF_QoSjK7MsL4-218PA.png)
圖檔來源 : [Deep-dive: Kubernetes NetworkPolicy in GKE]

NetworkPolicy 將流量分為
- Ingress: 簡單來說就是入站流量，任何對 Pod 發送流量都由  Ingress rule 判斷允許/拒絕該流量。

- Egress: 出站流量，任何由 Pod 內部發送出去的流量都由 Ingress rule 判斷允許/拒絕該流量。   
  > 📘 例如 Pod 中的應用程序連向其他 Pod、外部服務或資料庫，都屬於 Egress

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
  - `podSelector`: 透過 Label selector 指定這個 NetworkPolicy 套用在哪些 Pod 上。這樣可以控制該 NetworkPolicy 只影響具有特定標籤的 Pod。
  
  - `policyTypes`: 定義這個 NetworkPolicy 主要管理的流量類型。可以是 `Ingress`（進入流量）、`Egress`（離開流量），或兩者皆有。

  - `ingress.from`: 定義允許哪些入站流量（Ingress）。包括以下子欄位：
    - ``ipBlock``: 透過 `cidr` 指定允許哪些 IP 地址範圍進入 Pod。可以使用 `except` 排除某些 IP 區段，更精確地控制流量。
    
    - `namespaceSelector`: 透過 Label selector 允許來自具有指定 Label 的 namespace 中所有 Pod 的流量進入。
    
    - `podSelector`: 透過 Label selector 允許來自具有指定 Label 的 Pod 的流量進入。控制只有具有特定標籤的 Pod 能夠發送流量進入指定 Pod。
    - `ports`: 允許使用的 通訊協定 與 Port

  - `egress.to`: 定義了指定 Pod 能向哪裡發送流量。
    > 📘 `egress.to` 與 `ingress.from` 相同，都能透過 `ipBlock`、`namespaceSelector`、`podSelector`、`port` 指定允許出站流量（Engress）的目的地 與 通訊協定。

## 使用案例
透過上面的範例，我們知道能透過 Label selector 將 NetworkPolicy 套用到指定的 Pod 上那是不是我們能借鏡 [three-tier architecture] 將服務大致分為不同 tier，搭配 NetworkPolicy 降低資料洩漏的風險 
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240907/截圖-2024-09-07-下午10.36.16.70aaa5clyg.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240907/截圖-2024-09-07-下午10.36.16.70aaa5clyg.webp)

- `Frontend / Gateway tier`: 負責接受來自 Cluster 外部的流量，通常會負責處理認證授權，通過後將合法流量轉發到其他內部服務，此類常見的服務如 API Gateway、nginx 等
- `Backend tier`: 負責實現商業邏輯的服務，通常是一個 API 服務，如 Tomcat、Django。
- `Data tier`: 儲存資料的服務，如資料庫。

將上述概念轉為 NetworkPolicy 時，大致如下
- Default NetworkPolicy Rule: 不允許任何入站/出站流量。
- Frontend / Gateway tier NetworkPolicy Rule: 
  - `ingress`: 允許全部或特定外部流量進入該 tier 的 Pod
  - `egress` : 只允許將流量往 backend tier 發送
- Backend tier NetworkPolicy Rule:
  - `ingress`: 只允許 Frontend / Gateway tier 的流量傳入
  - `egress` : 只允許將流量往 backend、data tier 發送
- Data tier NetworkPolicy Rule:
  - `ingress`: 只允許 backend tier 的流量傳入
  - `egress` : 不允許任何出站流量

讓我們來一步一步進行配置
1. Default NetworkPolicy Rule
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

2. Frontend / Gateway tier NetworkPolicy Rule
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
兩個 NetworkPolicy 分別套用到 tier=frontend 或 gateway 的 Pod，允許任何流量進入，但只能將流量送往有 tier=backend 的 Pod，不可直接存取資料庫。

3. Backend tier NetworkPolicy Rule

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
允許 tier=frontend 或 gateway 的 Pod 傳入流量，避免 backend 服務被外部直接攻擊，且只能將流量送往同層的 Pod (tier=backend) 或 資料層的 Pod (tier=data)  的 Pod。

4. Data tier NetworkPolicy Rule:

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
僅允許 tier=backend 的 Pod 傳入流量，但不允許任何出站流量，減少資料被直接從內部搬出的風險。

# 注意事項
並不是每個 Kubernetes 都支援 NetworkPolicy，需要該 Kubernetes 使用的 CNI，否則配置的 NetworkPolicy，不會發揮作用也不會給出提示錯誤訊息，參考[官方文件](
https://kubernetes.io/docs/concepts/services-networking/network-policies/#prerequisites)
> 📘 kind 預設使用的 CNI 不支援 NetworkPolicy

# 小結
今天我們介紹了 NetworkPolicy 這個 Kubernetes 管理網路流量的資源，並參考傳統 three-tier 架構的配置來提升服務安全性，有興趣的讀者能依照需求配置 NetworkPolicy，例如
- 每個 Pod 只允許存取有依賴的 Pod 或外部服務，而不是個 tier 的 Pod。

明天的文章會帶讀者安裝 [cilium] 作為 CNI，並透過 [hubble] 來觀察網路流量。



# Refernce
- [Kubernetes 官方文件](https://kubernetes.io/zh-cn/docs/concepts/services-networking/network-policies/#networkpolicy-resource)
- [Deep-dive: Kubernetes NetworkPolicy in GKE]

[Deep-dive: Kubernetes NetworkPolicy in GKE]: https://medium.com/google-cloud/deep-dive-kubernetes-network-policy-in-gke-e9842ec6b1be

[橫向移動 (lateral movement)]: https://www.wiz.io/blog/lateral-movement-risks-in-the-cloud-and-how-to-prevent-them-part-2-from-k8s-clust
[three-tier architecture]: https://www.ibm.com/topics/three-tier-architecture

[cilium]: https://docs.cilium.io/en/stable/overview/intro/

[hubble]: https://docs.cilium.io/en/stable/overview/intro/#what-is-hubble