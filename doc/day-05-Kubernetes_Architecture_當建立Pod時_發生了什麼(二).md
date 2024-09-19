
# Day-05-Kubernetes Architecture 介紹 - 當建立 Pod 時，發生了什麼(二)

# 前言
昨天我們介紹了當使用 `kubectl` 指令建立 Pod 時，Control Plane 如何對請求進行驗證，並將配置持久化到資料庫中，接著透過調度演算法選擇合適的 Node，並將 Pod 與該 Node 綁定（Binding）。而今天我們將繼續探討 Worker Node 是如何將 Pod 真正建立出來的過程。



# Kubernetes Cluster Architecture
![Archtitecture](https://kubernetes.io/images/docs/kubernetes-cluster-architecture.svg)
圖檔來源 [Kubernetes 官方文件](https://kubernetes.io/docs/concepts/architecture/)

# kubelet
![https://miro.medium.com/v2/resize:fit:1400/format:webp/0*GWevN0yZS4roLOtu.png](https://miro.medium.com/v2/resize:fit:1400/format:webp/0*GWevN0yZS4roLOtu.png)    
圖檔來至: [itnext.io/what-happens-when-you-create-a-pod-in-kubernetes](https://itnext.io/what-happens-when-you-create-a-pod-in-kubernetes-6b789b6db8a8)

每個 Worker Node 上都會運行一個叫 kubelet 的 Process，負責監控和管理該 Node 上的 Pod。當 kube-apiserver 通知 kubelet 有新的 Pod 被綁定到該 Node 時，kubelet 會觸發 Pod 的建立流程。

該流程會透過使用 CRI、CNI、CSI 這三個 interface，將建立 Pod 的任務分派給底層的實現軟體。
1. **CRI (Container Runtime Interface)**：Kubernetes 用來與 Container Runtime 溝通，負責執行 Pod 的啟動、停止等操作。
    > 📘 常見的 Container Runtime 包括 containerd 和 CRI-O，這些軟體負責容器的建立、啟動、停止和銷毀。

2. **CNI (Container Network Interface)**：Kubernetes 用來管理 Pod 的網路，為 Pod 分配內部 IP，並確保 Pod 能與其他 Pod 以及外部世界進行網路通訊。

3. **CSI (Container Storage Interface)**：Kubernetes 用來管理存儲資源的生命週期，例如，當 Pod 需要 Volume 資源時，Kubernetes 會透過 CSI 將存儲空間與 Pod 綁定。

    > 📘 Kubernetes 透過這些標準接口，而不是依賴特定的實作，來完成 Pod 的建立和管理，使得 Kubernetes 管理者可以根據需求替換不同的實現軟體。

簡單來說，Kubernetes 透過：
- CRI 初始化並啟動 Pod 的容器
- CNI 為 Pod 配置網路並分配內部 IP
- CSI 掛載 Volume 為 Pod 提供存儲空間

當這些步驟完成後，Pod 及其容器便正式在該 Worker Node 上運行。

![https://miro.medium.com/v2/resize:fit:1400/format:webp/0*WgOaCA0trzf4SmjJ.png](https://miro.medium.com/v2/resize:fit:1400/format:webp/0*WgOaCA0trzf4SmjJ.png)
圖檔來至: [itnext.io/what-happens-when-you-create-a-pod-in-kubernetes](https://itnext.io/what-happens-when-you-create-a-pod-in-kubernetes-6b789b6db8a8)

最後 `kubelet` 會將該 Pod 的詳細資訊(IP、狀態) 回報到 Control Plane (透過 `kube-apiserver` 儲存到 etcd)。

到這裡，這個 Pod 的初始化過程基本上算是完成了。如果該 Pod 中的容器運行正常，對於長期運行的 Pod（如 Deployment），Pod 的狀態會變為 `Running`。而對於一次性任務的 Pod（如 Job），當所有容器成功完成任務後，Pod 的狀態會變為 `Succeeded`。

![https://miro.medium.com/v2/resize:fit:1400/format:webp/0*vg1x1jEQ8pWyNzu9.png](https://miro.medium.com/v2/resize:fit:1400/format:webp/0*vg1x1jEQ8pWyNzu9.png)

若在執行 `kubectl get pod` 時，發現 Pod 的狀態為：

- **ImagePullBackOff**：這代表 CRI 無法取得該 Pod 定義的 container image。建議檢查以下幾點：
  - 檢查 image 的位址和 tag 是否正確
  - 確認是否有權限從該 registry 拉取 container image
  - 檢查與 image registry 的網路連線是否正常

- **CrashLoopBackOff**：這代表 Pod 的容器持續崩潰。可以透過 `kubectl describe pod {pod-name}` 檢查具體原因，通常可能是：
  - 容器內的應用程式啟動失敗
  - 使用 `kubectl logs {pod-name}` 查看容器的運行日誌
  - 檢查 liveness probe 是否與應用程式提供的端點相符

## 小結
整個 Pod 部署的完整流程可以參考下圖，讓你快速了解 Kubernetes 的各個組件如何協同合作來完成 Pod 的部署：
![https://miro.medium.com/v2/format:webp/1*WDJmiyarVfcsDp6X1-lLFQ.png](https://miro.medium.com/v2/format:webp/1*WDJmiyarVfcsDp6X1-lLFQ.png)
圖片來源：[The journey of a Pod: A guide to the world of Pod Lifecycle](https://medium.com/@seifeddinerajhi/navigating-the-journey-of-a-pod-a-guide-to-the-exciting-world-of-pod-lifecycle-a1fbc2c98c55)

如果你對 `kubelet` 如何與 CRI、CNI 互動感興趣，可以參考下圖，進一步了解它們之間的運作原理：
![https://miro.medium.com/v2/resize:fit:1400/1*OuXfcIUU-VShb3kXmEU2BA.png](https://miro.medium.com/v2/resize:fit:1400/1*OuXfcIUU-VShb3kXmEU2BA.png)
圖片來源：[The birth story of the kubernetes pods](https://sitereliability.in/deep-dive-the-birth-of-a-kubernetes-pod-understand-the-kubernetes-internals)

到這裡，大家可能會發現 `kube-proxy` 這個 Worker Node 的核心組件，並沒有在上述流程中出現。明天我們將深入介紹 `kube-proxy`，這個在 Kubernetes 中不可或缺的組件，看看它如何在網路流量轉發中發揮作用。


# Refernce
- [itnext.io/what-happens-when-you-create-a-pod-in-kubernetes]
- [The birth story of the kubernetes pods]
- [kubectl 创建 Pod 背后到底发生了什么]
- [The journey of a Pod: A guide to the world of Pod Lifecycle]

[itnext.io/what-happens-when-you-create-a-pod-in-kubernetes]:
https://itnext.io/what-happens-when-you-create-a-pod-in-kubernetes-6b789b6db8a8

[The birth story of the kubernetes pods]: https://sitereliability.in/deep-dive-the-birth-of-a-kubernetes-pod-understand-the-kubernetes-internals

[kubectl 创建 Pod 背后到底发生了什么]: https://icloudnative.io/posts/what-happens-when-k8s/

[ResourceQuota]:https://kubernetes.io/docs/concepts/policy/resource-quotas/

[LimitRanger]: https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/

[init-container]: https://kubernetes.io/docs/concepts/workloads/pods/init-containers/

[The journey of a Pod: A guide to the world of Pod Lifecycle]: https://medium.com/@seifeddinerajhi/navigating-the-journey-of-a-pod-a-guide-to-the-exciting-world-of-pod-lifecycle-a1fbc2c98c55

[liveness]: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/