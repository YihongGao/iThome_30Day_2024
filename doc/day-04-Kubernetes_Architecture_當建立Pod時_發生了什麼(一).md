
# Day-04-Kubernetes Architecture 介紹 - 當建立 Pod 時，發生了什麼(一)

# 前言
前幾天我們深入了解了 Kubernetes 的核心組件，今天要來聊聊當我們 **向 Kubernetes 發出建立 Pod 的請求** 時，背後到底發生了哪些事情，Pod 是如何被建立出來的。透過認識這個過程，希望能讓讀者在開發或維護 Kubernetes 時，更加順暢且安心。


# Kubernetes Cluster Architecture
![Archtitecture](https://kubernetes.io/images/docs/kubernetes-cluster-architecture.svg)
圖檔來源 [Kubernetes 官方文件](https://kubernetes.io/docs/concepts/architecture/)

# kubectl
當你使用 kubectl 指令時，kubectl 這個指令行工具會在 Client 端進行以下任務：

基本檢核：kubectl 會在本地進行一些基本的檢查，例如語法錯誤或無效的資源名稱，將明顯不會成功的請求在 Client 端返回錯誤訊息，減少 kube-apiserver 的壓力。此外，kubectl 會將可用的資源版本規範，使用 OpenAPI 格式快取在本地 ~/.kube/cache 目錄中，並在操作時根據這些規範進行初步檢查。

客戶端身份憑證：kubectl 會根據以下優先順序來尋找身份憑證，用於後續與 kube-apiserver 進行身份驗證：
- 命令列參數：若明確指定了憑證（如 --kubeconfig、--certificate-authority 等），kubectl 會優先使用這些參數指定的憑證。
- 環境變數：如果設置了 KUBECONFIG 環境變數，kubectl 會使用該變數指定的配置檔案來尋找憑證。
- 預設 kubeconfig 檔案：若未設置 KUBECONFIG 變數，kubectl 會使用 ~/.kube/config 檔案中的預設配置來尋找憑證。

將指令封裝成 Http 請求：kubectl 會將操作封裝成 HTTP 請求，透過 RESTful API 發送至 kube-apiserver，完成對 Kubernetes 資源的操作。

如同 [Day-02-Kubernetes Architecture 介紹 - Control Plane](https://ithelp.ithome.com.tw/articles/10347299) 所述，所有的 Kubernetes 資源操作都必須通過 kube-apiserver 提供的 RESTful API 來進行，kubectl 也不例外。

# 建立 Pod 的流程
## 1. Http 請求從 kubectl 送出 (in Client)
![https://miro.medium.com/v2/resize:fit:1400/format:webp/0*01aDK3UDdeWxxM5V.png](https://miro.medium.com/v2/resize:fit:1400/format:webp/0*01aDK3UDdeWxxM5V.png)
圖檔來至: [itnext.io/what-happens-when-you-create-a-pod-in-kubernetes](https://itnext.io/what-happens-when-you-create-a-pod-in-kubernetes-6b789b6db8a8)

如同先前說明的，kubectl 會透過 OpenAPI 格式找到適合的 API enpint 並進行格式校驗後，將 身份憑證 與 操作資源 的 payload 封裝成 Http Request 送往 `kube-apiserver` 

## 2. 認證 和 授權、Admission controllers  (kube-apiserver)
![https://miro.medium.com/v2/resize:fit:1400/format:webp/0*tYNAp-JiOoF7-frD.png](https://miro.medium.com/v2/resize:fit:1400/format:webp/0*tYNAp-JiOoF7-frD.png)
圖檔來至: [itnext.io/what-happens-when-you-create-a-pod-in-kubernetes](https://itnext.io/what-happens-when-you-create-a-pod-in-kubernetes-6b789b6db8a8)

當 kube-apiserver 收到 HTTP Request 請求後，會首先從中取得身份憑證，並進行身份認證（Authentication），確認該用戶是否為此 Cluster 的合法用戶。接著，進行授權（Authorization），檢查該用戶是否具備操作該資源的權限。

在通過認證（Authentication）和授權（Authorization）後，請求還必須通過 Admission controllers 的檢查。Admission controllers 是一組處理鏈，它們具備兩種主要功能：

- 調整請求資源的值（Mutating Admission Controllers）
- 檢查請求資源的值是否正確（Validating Admission Controllers）。如果檢查未通過，則會拒絕該請求並返回錯誤。

以下是常見的 Admission controllers：

- **ResourceQuota**：當 Pod 請求的資源超過 Namespace 中設定的資源配額時，請求會被拒絕。
- **LimitRanger**：當 Pod 未指定資源請求時，將給予預設的資源值；若 Pod 請求的資源超出 LimitRanger 配置的限制，請求也會被拒絕。

## 3. 將資料持久化到 etcd (kube-apiserver)
終於通過層層檢查，讓建立 Pod 的請求內容，送到了 Kubernetes 的資料庫：`etcd`。
然而 Pod 尚未真的被建立出來，而是處於 `Pending` 的狀態，代表 Scheduler 尚未幫這個 Pod 找到適合的家。
> 📘 關於 Pod 的狀態機，能參考[官方文件](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-phase)

## 4. 幫 Pod 找適合的家 (scheduler)
![https://miro.medium.com/v2/resize:fit:1400/format:webp/0*vQ-KQONXaQ_t4EQ2.png](https://miro.medium.com/v2/resize:fit:1400/format:webp/0*vQ-KQONXaQ_t4EQ2.png)
圖檔來至: [itnext.io/what-happens-when-you-create-a-pod-in-kubernetes](https://itnext.io/what-happens-when-you-create-a-pod-in-kubernetes-6b789b6db8a8)

當 scheduler 從 kube-apiserver 發現有處於 Pending 狀態的 Pod 時，它會進行兩個主要階段的操作：
- **過濾**：scheduler 會依據 Pod 的需求（如 Affinity 和 anti-Affinity 等配置）以及 Node 的資源狀況，過濾出一個符合條件的 Node 列表。
- **評分**：對於篩選出的 Node，scheduler 會根據多種因素（如資源利用率、網路延遲等）進行評分，然後選出得分最高的 Node。

最後，scheduler 會使用 kube-apiserver 的 API 將 Pod 與選中的 Node 綁定（Binding）。

📘 這裡的綁定（Binding）是指 scheduler 更新 Pod 的 `spec.nodeName` 欄位。scheduler 並不是負責實際建立 Pod 的組件，Pod 的實際啟動由該 Node 上的 kubelet 負責。

到目前為止，這個 Pod 的資訊仍只是一筆存在 etcd 中的資料，尚未有任何 container 因為這個操作而被啟動。


## 小結
今天介紹了 建立 Pod 旅程的前半段，這些事件大多都發生在 Control Plane 中
- 從 Client 端使用 `kubectl` 指令送出建立 Pod 的請求到 `kube-apiserver`
- `kube-apiserver` 中進行 認證、授權、Admission controllers 校驗，最終持久化到 etcd。
- `scheduler` 依據調度策略找到最適合該 Pod 的 Node。

明天會繼續介紹，建立 Pod 旅程的後半段，關於 worker node 的組件如何接手處理這筆 Pod 的資料。

# Refernce
- [itnext.io/what-happens-when-you-create-a-pod-in-kubernetes]
- [The birth story of the kubernetes pods]
- [kubectl 创建 Pod 背后到底发生了什么]

[itnext.io/what-happens-when-you-create-a-pod-in-kubernetes]:
https://itnext.io/what-happens-when-you-create-a-pod-in-kubernetes-6b789b6db8a8

[The birth story of the kubernetes pods]: https://sitereliability.in/deep-dive-the-birth-of-a-kubernetes-pod-understand-the-kubernetes-internals

[kubectl 创建 Pod 背后到底发生了什么]: https://icloudnative.io/posts/what-happens-when-k8s/

[ResourceQuota]:https://kubernetes.io/docs/concepts/policy/resource-quotas/

[LimitRanger]: https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/