
# Day-11 KEDA 介紹 - 基於 Prometheus 自動擴展 Pod

# 前言
昨天已經將 KEDA 安裝到 Kubernetes 中，今天會來安裝 Prometheus，並使用 KEDA 依據 Prometheus 中的 metrics 配置自動擴展策略。

# 環境準備
需要再 Kubernetes 中進行以下準備
- 安裝 prometheus 
- 部署 workload
- 收集 workload 的 metrics

## 安裝 prometheus
> 📖 若讀者已有 Prometheus 並且能收集到 Pod 的 metrics 時，能跳過此步驟
為了方便起見，透過 [Kube-Prometheus] 安裝 prometheus，參考[官方文件](https://prometheus-operator.dev/docs/getting-started/installation/#install-using-kube-prometheus)步驟進行安裝

```yaml
git clone https://github.com/prometheus-operator/kube-prometheus.git
kubectl create -f kube-prometheus/manifests/setup -f kube-prometheus/manifests
```
安裝完成後，能在 namespace: `monitoring` 中看到 Prometheus、Grafana 相關組件。
首先我們需要微調幾個地方
1. 給 prometheus 服務使用的 service account 能 get 其他 namespace 的權限，才能跨 namespace 收集 metrics
    ```shell
    kubectl edit clusterrole prometheus-k8s 
    ```
    再 `rules:` 中添加 read pods 的權限
    ```yaml
    - apiGroups:
    - ""
    resources:
    - pods
    verbs:
    - get
    - list
    - watch
    ```

## 部署 workload
我們部署一個有 export prometheus 格式 metrics 的 spring boot Application
```yaml
# keda-demo.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: keda-demo
    scrape: spring-boot-prometheus-exporter
  name: keda-demo
  namespace: ithome
spec:
  replicas: 1
  selector:
    matchLabels:
      app: keda-demo
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: keda-demo
        # 透過此 label 讓 PodMonitor 發現，並收集 metrics
        scrape: spring-boot-prometheus-exporter
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: keda-demo
        matchLabelKeys:
        - pod-template-hash
      containers:
      - image: luciferstut/spring-boot-application-for-ithome2024:day10
        imagePullPolicy: Always
        name: spring-boot-application-for-ithome2024
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        ports:
        - name: web
          containerPort: 8080
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          failureThreshold: 3
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          failureThreshold: 3
          periodSeconds: 10
          
---

apiVersion: v1
kind: Service
metadata:
  labels:
    app: keda-demo
    scrape: spring-boot-prometheus-exporter
  name: keda-demo
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: keda-demo
```
部署該 Deployment 後，來看一下有什麼 metrics 
```shell
kubectl apply -f keda-demo.yml

kubectl exec -it ${pod-name} -- curl localhost:8080/actuator/prometheus
```
能看到其中一個 Metrics 叫 `http_server_requests_seconds_count`，它代表每秒的 http request 數量。


## 收集 workload 的 metrics
1. 建立 PodMonitor 來配置收取 metrics 的 endpoint
```yaml
# pod-monitor.yml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: spring-boot-prometheus-exporter
  namespace: ithome
  labels:
    scrape: spring-boot-prometheus-exporter
spec:
  selector:
    matchLabels:
      scrape: spring-boot-prometheus-exporter
  podMetricsEndpoints:
  - port: web
    path: /actuator/prometheus
```

2. 配置 prometheus 處理哪些 PodMonitor
```shell
kubectl label namespaces ithome monitoring=true     

kubectl patch prometheus k8s \
  --namespace monitoring \
  --type='merge' \
  --patch='{"spec": {"podMonitorSelector": {"matchLabels": {"scrape": "spring-boot-prometheus-exporter"}}, "podMonitorNamespaceSelector": {"matchLabels": {"monitoring": "true"}}}}'
```

到這裡，理論上我們已經完成環境配置，Kubernetes 中應該運行了至少以下組件
- KEDA (於 Day10 安裝)
- Prometheus
- PodMonitor 配置如何收取 Metrics
- 1 個 Deployment(`keda-demo`) 運行著 spring-boot application

能透過 port-forward 進入 prometheus 檢查一下是否正確收到 spring-boot application 的 metrics
```shell
kubectl port-forward -n monitoring svc/prometheus-k8s 9090:9090
```
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240901/截圖-2024-09-01-下午10.07.28-1.7zqdeptcvi.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240901/截圖-2024-09-01-下午10.07.28-1.7zqdeptcvi.webp)

使用 `http_server_requests_seconds_count` 應該要能查到 spring-boot application 的 metrics

# 設定 KEDA 依據 HTTP QPS 來進行擴展

```yaml
# keda-scaled-object.yml 
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: keda-demo
  namespace: ithome
spec:
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus-k8s.monitoring.svc.cluster.local:9090
        query: sum(rate(http_server_requests_seconds_count{container="spring-boot-application-for-ithome2024", "namespace"="ithome"}[2m]))
        threshold: '10'        
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: keda-demo
  minReplicaCount: 2
  maxReplicaCount: 6
  fallback:
    failureThreshold: 2
    replicas: 6 
```

**ScaledObject** 是 KEDA 提供的 CRD，用來定義 workload 的擴展策略與事件來源，以下介紹幾個重要的欄位：
- `triggers`：定義事件來源的區塊，以下以 `prometheus`` 為例：
    - `serverAddress`：Prometheus 的 URL，KEDA 會透過此地址獲取 metrics。
    - `query`：Prometheus 查詢語法，這裡使用 `http_server_requests_seconds_count` 來計算 API QPS。
    - `threshold`：KEDA 根據 query 返回的值，超過此閥值時觸發擴/縮容操作。

- `scaleTargetRef` ：定義要操作擴/縮容的 Kubernetes Workload，例如 Deployment。

- `minReplicaCount / maxReplicaCount` ：定義 Pod 副本的最小與最大數量，與 HPA 的設定類似。

- `fallback`：當 triggers 無法使用時的應對方案：
    - `failureThreshold`：當 `trigger` 無法正常工作且超過此失敗次數時，啟用 `fallback`
    - `replicas`：當 `trigger` 無法使用時，固定的 Pod 副本數量。

在這個範例中，`ScaledObject` 使用 Prometheus 來決定擴/縮容條件。當 API QPS 超過 10 時，KEDA 會對名為 `keda-demo` 的 Deployment 進行擴容操作；當 QPS 低於閥值時，則進行縮容。Pod 副本數量最少為 2 個，最多可達 6 個。如果 Prometheus 無法使用，KEDA 會將 Pod 副本數固定為 6 個。

我們來部署 ScaledObject 並對 Deployment 給予一些 http 請求使其擴容
```shell
# 部署 ScaledObject
kubectl apply -f keda-scaled-object.yml 

# 啟動一個 pod 持續向 keda-demo 的 pod 發出 http request 來觸發擴容
kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://keda-demo:8080/greeting?name=ithome; done"
```

能從 prometheus 看到 QPS metrics 陸續提升，而 Pod 的數量也逐漸從 2 提升到 6 個 pod
```shell
kubectl get pod -o wide
NAME                         READY   STATUS    RESTARTS   AGE     IP            NODE                  NOMINATED NODE   READINESS GATES
keda-demo-55544b4fbb-9pslh   1/1     Running   0          5m9s    10.244.4.62   ithome-2024-worker3   <none>           <none>
keda-demo-55544b4fbb-bjhzt   1/1     Running   0          4m35s   10.244.2.49   ithome-2024-worker    <none>           <none>
keda-demo-55544b4fbb-c48wt   1/1     Running   0          110s    10.244.4.67   ithome-2024-worker3   <none>           <none>
keda-demo-55544b4fbb-dqd98   1/1     Running   0          5m6s    10.244.2.47   ithome-2024-worker    <none>           <none>
keda-demo-55544b4fbb-l5c4c   1/1     Running   0          4m51s   10.244.4.63   ithome-2024-worker3   <none>           <none>
keda-demo-55544b4fbb-ncpbg   1/1     Running   0          4m51s   10.244.2.48   ithome-2024-worker    <none>           <none>
```

# 小結
今天我們透過 KEDA 使用 prometheus 作為 自動擴/縮容的策略依據，突破了只能依賴 CPU 和記憶體的限制，提供了更多靈活運用的空間。例如，我們可以根據 API 的 Response time 等 metrics 進行擴/縮容，進一步提升資源管理的靈活性。

明天我們將繼續介紹更多 KEDA 的應用案例。

# Refernce
- [KEDA]

[KEDA]: https://keda.sh/
[HPA]: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/
[HPA (Horizontal Pod Autoscaling)]: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/

[2023/day-29-Kubernetes 介紹-Pod 水平自動伸縮 (Horizontal Pod Autoscaler)]: https://ithelp.ithome.com.tw/articles/10336846

[kind]: https://kind.sigs.k8s.io/

[helm]: https://helm.sh/

[Kube-Prometheus]: https://prometheus-operator.dev/docs/getting-started/installation/