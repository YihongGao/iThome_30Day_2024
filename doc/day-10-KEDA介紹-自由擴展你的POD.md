
# Day-10 KEDA 介紹 - 自由擴展你的 Pod

# 前言
我們已了解多種管理 Pod 副本分佈來保持服務高可用的方法，若再結合 [HPA (Horizontal Pod Autoscaling)] 的功能，便能根據 Pod CPU / Memory 自動擴/縮容 Pod 副本，達到系統的高可用性與良好的資源利用率。
> 📘 HPA 的詳細使用方式可參閱官方文件或 [2023/day-29-Kubernetes 介紹-Pod 水平自動伸縮 (Horizontal Pod Autoscaler)]

然而，[HPA] 存在一些限制：
- **Metrics 選擇少** : 原生 HPA 只能透過 Container CPU, Memory 來配置擴/縮容條件，當關鍵指標為其他數據時，則無法有效調整 Pod 副本。
- **反應速度較慢**：當 CPU, Memory 升高，可能都是收到大量負載，而導致資源使用率提高的現象。
- **無法縮容到 0 個 Pod**：無法更有效的利用資源

[KEDA]（Kubernetes Event-Driven Autoscaling）是 CNCF 的一個專案，它能增強 HPA 的功能，提供更靈活的擴/縮容能力，使 Kubernetes 能夠根據更多類型的指標進行自動調整，並實現更高效的資源管理。

# 什麼是 KEDA (Kubernetes Event-driven Autoscaling)
[KEDA] 故名思義，是一個基於 Event-driven 的 自動擴/縮容組件，它允許使用多種外部服務作為事件來源，從而更靈活地控制 Pod 的擴/縮容，適用於多樣化的場景。例如：
- **使用 Prometheus 中的 metrics 作為事件來源**：     
    - 當 API QPS(query per second) 超過閥值時，進行擴/縮容。
    - 平均 Response time 超過閥值時，進行擴/縮容
- **使用各種 Message Queue 作為事件來源**：
    - Queue 長度大於閥值時，進行擴/縮容
- **使用 Database 作為事件來源**：
    - 依照 SQL 結果進行擴/縮容

# KEDA 架構
![https://keda.sh/img/keda-arch.png](https://keda.sh/img/keda-arch.png)

在 Kubernetes 中，KEDA 透過以下三個主要組件與 HPA 協同合作來實現自動擴/縮容的功能。

## 主要組件
1. **Controller** : 根據配置判斷 Workload 是否需要啟動，啟動時將副本數調整為 1，關閉時將副本數調整為 0。
> 📘 KEDA Controller 僅負責 Pod 副本數的 0 -> 1 或 1 -> 0 的擴/縮容，其餘的擴/縮容（如 1 -> n 或 n -> 1）由 HPA 利用 external metrics 處理。

2. **Metrics Adapter** : 將外部事件轉為 metrics 提供給 HPA 作為 external metrics，進行進一步擴/縮容決策。

3. **Admission Webhooks**：用於驗證 Kubernetes 資源的變更，防止配置錯誤。

簡而言之，KEDA 會監控外部服務（如 Prometheus）提供的 metrics，透過 Metrics Adapter 將這些 metrics 傳遞給 Controller 和 HPA，依據指標數據來動態調整 Workload 的副本數。

## CustomResourceDefinition（CRD)
1. **ScaledObjects**：
    - 用來定義 Kubernetes Workload（如 Deployment 或 StatefulSet）的自動擴展策略
    - 包含監控的事件來源及觸發擴展的條件等配置。

2. **ScaledJob**：
    與 ScaledObjects 相似，但管理的是 Kubernetes Job 的擴展策略。

3. **TriggerAuthentication**：
    - 用來配置在存取事件來源時攜帶的認證資訊（如 API Key），供命名空間內的 ScaledObjects 和 ScaledJob 使用。

4. **ClusterTriggerAuthentication**：
    - 與 TriggerAuthentication 相似，但適用於整個 Cluster 中的 ScaledObjects 和 ScaledJob。

# 環境準備
這幾天我們會在本地的 [kind]，安裝 [KEDA] 並進行實驗其功能性。

## 透過 [helm] 安裝 [KEDA]
```shell
helm repo add kedacore https://kedacore.github.io/charts

helm repo update

helm install keda kedacore/keda --namespace keda --create-namespace
```

能 kubernetes 中建立了一個新的 namespace: `keda`，裡面運作了三個 Deployment，分別就是 KEDA 提供服務三個主要組件
```shell
kubectl get deployments.apps -n keda 
NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
keda-admission-webhooks           1/1     1            1           21h
keda-operator                     1/1     1            1           21h
keda-operator-metrics-apiserver   1/1     1            1           21h
```

# 小結
明天我們會介紹 KEDA 的 CRD，並使用 Prometheus 的 metrics 當作擴展策略。

# Refernce
- [KEDA]
- [KEDA — 基本介紹](https://medium.com/@felix0607/keda-%E5%9F%BA%E6%9C%AC%E4%BB%8B%E7%B4%B9-2c7bf8249033)


[KEDA]: https://keda.sh/
[HPA]: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/
[HPA (Horizontal Pod Autoscaling)]: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/

[2023/day-29-Kubernetes 介紹-Pod 水平自動伸縮 (Horizontal Pod Autoscaler)]: https://ithelp.ithome.com.tw/articles/10336846

[kind]: https://kind.sigs.k8s.io/

[helm]: https://helm.sh/