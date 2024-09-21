# Day-09-Kubernetes 如何調度你的 Pod - Pod Topology Spread Constraints

# 前言
前兩天我們介紹了
- NodeSelector
- Node Affinity / Anti-affinity
- Inter-Pod Affinity / Anti-affinity 

已經能靈活地引導 `scheduler` 幫我們把 Pod 調度到我們期待的 Worker Node 了。
但也發現若希望 Pod 平均分散到 Worker Node，仍然會遇到一些問題
- 當僅 Node Affinity 時，Pod 仍可能集中在少數 Node 上
- 搭配 Inter-Pod Affinity 之後，限制每個 Node 只能一個同類的 Pod 時，又降低最大可用的副本數量。

這兩種問題可能會成為降低可用性的因素，所以今天要介紹的 `Pod Topology Spread Constraints` 就是專門解決此問題而生的設計的。

# Pod Topology Spread Constraints
`Pod Topology Spread Constraints`` 的目的是將 Pod 儘可能均勻地分散到不同的 **topology** 中，例如不同的 Node、Region 或 Zone。其核心概念是 **skew**，即每組 Pod 在不同 topology 中的分布差異。

### 計算公式為
skew = Pods **number** matched in **current** topology - **min** Pods matches in a topology

![https://www.hwchiu.com/assets/images/SkWAL7S33-08bbfe0f59afad8380f63e1af86df610.png](https://www.hwchiu.com/assets/images/SkWAL7S33-08bbfe0f59afad8380f63e1af86df610.png)
圖檔來源: [HWCHIU 學習筆記 / 解密 Assigning Pod To Nodes(下)]

以上圖為例，有三個 topology 分別運行著 3、2、1 個 Pod，因此可以計算出最少 Pod 數量 min Pods matches in a topology 為 1。每個topology的 skew 值如下：

- **min** Pods matches in a topology : `1`
1. Topology A
    - Pods **number** matched in **current** topology: `3`
    - skew: `2 (3 - 1)`
2. Topology B
    - Pods **number** matched in **current** topology: `2`
    - skew: `1 (2 - 1)`
2. Topology C
    - Pods **number** matched in **current** topology: `1`
    - skew: `0 (1 - 1)`

瞭解 skew 計算方式後，Pod Topology Spread Constraints 的配置變得更為直觀。

```yaml
spec:
  topologySpreadConstraints:
  - topologyKey: <string>
    maxSkew: <integer>
    labelSelector: <object>
    whenUnsatisfiable: <string>
    
```
- `topologyKey`: 定義按哪個 Node Label 來劃分topology，類似於 Inter-Pod Affinity 的用法。
- `maxSkew`: 每個 topology 中的最大 skew 限制，即 Pod 分布不均勻的容忍度。
- `labelSelector`: 指定哪些 Pod 需要參與計算。
- `whenUnsatisfiable`: 當全部的 topology 都不滿足 `maxSkew` 的條件時的處理方式
    - `DoNotSchedule`: (Default) 不調度該 Pod，該 Pod 會處於 Pending
    - `ScheduleAnyway`: 仍然調度該 Pod，並優先選擇 Skew 低的 topology。

我們來看一個使用範例
![https://kubernetes.io/images/blog/2020-05-05-introducing-podtopologyspread/api.png](https://kubernetes.io/images/blog/2020-05-05-introducing-podtopologyspread/api.png)
圖檔來自: [Kubernetes 官方](https://kubernetes.io/blog/2020/05/introducing-podtopologyspread/)

當我們 或 [HPA(Horizontal Pod Autoscaling)] 要新增一個 Pod(label `app=foo`) 的副本時，目前兩個 topoloy 中該 Pod 的分佈為 `2,0` (zone2 中的 Pod label 不符合條件，故不算在內)。
故 `scheduler` 會計算當把 Pod 調度到該 topology 時，是否超過 `maxSkew` 的配置，依此為例
- 若調度到 zone1 的 topology 時，skew 為 `3`，不符合 `maxSkew: 1` 的條件，故不可調度到該 topology
- 若調度到 zone2 的 topology 時，skew 為 `0`，符合 `maxSkew: 0` 的條件，故可以調度到該 topology

與 `Inter-Pod Affinity` 相比較，`Pod Topology Spread Constraints` 提供的 `maxSkew` 與 `whenUnsatisfiable`，讓 Pod 能均勻分佈到每個 topology，且不再有每個 topology 只能運行一個 Pod 的限制，大大的提高資源利用率。

## 進階用法
在某些情境下，Pod 的分佈可能與 `maxSkew` 設定不符，特別是在 Deployment 進行 Rolling Update 時，會出現新舊版本的 Pod 同時被 `LabelSelectors` 選中的情況。
![https://miro.medium.com/v2/resize:fit:1400/format:webp/0*DSX-uCIKI-MlW92B](https://miro.medium.com/v2/resize:fit:1400/format:webp/0*DSX-uCIKI-MlW92B)
圖檔來自: [Avoiding Kubernetes Pod Topology Spread Constraint Pitfalls]

假設目前 Pod 均勻分佈在三個topology中，其中一個 Pod 為舊版本。因為三個 topology 的 `skew` 值皆為 0，新的 Pod 可以被部署到任何一個 topology。這可能導致以下情況：

![https://miro.medium.com/v2/resize:fit:1400/format:webp/0*nCdvNNKaerBoJLFg](https://miro.medium.com/v2/resize:fit:1400/format:webp/0*nCdvNNKaerBoJLFg)
圖檔來自: [Avoiding Kubernetes Pod Topology Spread Constraint Pitfalls]

例如，第三個新版本的 Pod 被調度到 AZ1，而 AZ3 的舊版本 Pod 在完成 Rolling Update 後被 Terminated，最終導致 Pod 分佈變為 `2,1,0`，違反了 `maxSkew: 1` 的限制。

為了避免這種情況，應該使用 **Pod Topology Spread Constraints** 的 `matchLabelKeys` 屬性，搭配 Deployment 自動為 Pod 添加的 `pod-template-hash` label。
> 📘 每當 Deployment 的 `spec.template` 更改時，都會得到一個 hash 值，Pod 上能透過 `pod-template-hash` label 獲取該 hash 值。

Deployment yaml 片段 範例如下：
```yaml
spec:
  containers:
    - name: nginx
      image: nginx:latest
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: nginx
    matchLabelKeys:
    - pod-template-hash
```

當在計算 `skew` 時，因 `pod-template-hash` label 的值不同，會將新舊版本的 Pod 分開計算 `skew`。回到 Rolling Update 的例子，這次的計算結果如下：

![https://miro.medium.com/v2/resize:fit:1400/format:webp/0*DSX-uCIKI-MlW92B](https://miro.medium.com/v2/resize:fit:1400/format:webp/0*DSX-uCIKI-MlW92B)
圖檔來自: [Avoiding Kubernetes Pod Topology Spread Constraint Pitfalls]

這次計算如下
 - **min** Pods matches in a topology : `0` (因為舊版本的不計算在內，而 AZ3 無任何匹配的 Pod，因此為 0)
1. Topology A
    - Pods **number** matched in **current** topology: `1`
    - skew: `1 (1 - 0)`
2. Topology B
    - Pods **number** matched in **current** topology: `1`=
    - skew: `1 (1 - 0)`
2. Topology C
    - Pods **number** matched in **current** topology: `0`
    - skew: `0 (0 - 0)`

**因此，第三個新版本的 Pod 將被正確地均勻分佈到 AZ3。**

# 小結
`Pod Topology Spread Constraints` 提供了一個有效的機制，確保 Pod 的均勻分佈。透過設定 `maxSkew` 和 `topologyKey`，避免 Pod 過度集中於特定topology 內，進而提升應用的高可用性與資源利用效率。

此外，配合 `matchLabelKeys` 和 `pod-template-hash`，可以避免新舊版本的 Pod 被誤算在同一 topology 中，進一步保證更新過程中的分佈一致性與資源平衡。這樣的機制不僅增強了 Kubernetes 調度的靈活性，也確保了部署時的穩定性與可擴展性。

# Refernce
- [kubernetes 官方文件](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Avoiding Kubernetes Pod Topology Spread Constraint Pitfalls]
- [HWCHIU 學習筆記 / 解密 Assigning Pod To Nodes(下)]
- [小信豬的部落格 / [Kubernetes] Assigning Pods to Nodes](https://godleon.github.io/blog/Kubernetes/k8s-Assigning-Pod-to-Nodes/)



[kind]: https://kind.sigs.k8s.io/

[HWCHIU 學習筆記 / 解密 Assigning Pod To Nodes(下)]: https://www.hwchiu.com/docs/2023/k8s-assigning-pod-2

[HPA(Horizontal Pod Autoscaling)]: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/

[Avoiding Kubernetes Pod Topology Spread Constraint Pitfalls]: https://medium.com/wise-engineering/avoiding-kubernetes-pod-topology-spread-constraint-pitfalls-d369bb04689e