
# Day-13 KEDA 介紹 - 基於 Kubernetes Resource 擴展 Pod

# 前言
前兩天我們介紹了如何透過 Prometheus 和 Message Queue 來配置自動擴展策略。但有時候我們需要使用更簡單的策略，例如 **根據上游 Pod 的數量來進行擴展**。理論上，當上游的 Pod 進行擴展時，下游 Pod 也可能會受到更多流量影響。因此，若上游服務的自動擴展策略已經完善，下游服務可以依據上游 Pod 的數量來同步進行擴展。

# 環境準備
需要再 Kubernetes 中進行以下準備，來模擬上下游服務
- 部署 nginx 模擬上游服務，並有 2 個 Pod 副本
- 部署 redis 模擬下游服務，並有 1 個 Pod 副本

## 部署 nginx 模擬上游服務，並有 2 個 Pod 副本
> 代碼能參閱 [GitHub](https://github.com/YihongGao/iThome_30Day_2024/tree/main/resources/day13)

```yaml
# nginx.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
```
部署 nginx
```shell
kubectl apply -f nginx.yml
```

## 部署 redis 模擬下游服務，並有 1 個 Pod 副本
```yaml
# redis.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  selector:
    matchLabels:
      app: redis
  replicas: 1
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:latest
```
部署 nginx
```shell
kubectl apply -f redis.yaml
```

這時能看到 Kubernetes 中有 3 個 Pod 在運行
- 2 個 nginx Pod
- 1 個 redis Pod
```shell
kubectl get pod
NAME                     READY   STATUS    RESTARTS   AGE
nginx-7584b6f84c-5bqlm   1/1     Running   0          8m7s
nginx-7584b6f84c-bmhhv   1/1     Running   0          13m
redis-644585c74b-8v5rp   1/1     Running   0          13m
```
# 配置 KEDA
來設定 `ScaledObject` 使 Nginx 的 Pod 數量作為 Redis 擴展的依據。

## 配置 ScaledObject
```yaml
# keda-scaled-object.yml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: keda-kubernetes-workload-demo
  namespace: ithome
spec:
  minReplicaCount: 1
  maxReplicaCount: 4
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: redis
  triggers:
  - type: kubernetes-workload
    metadata:
      podSelector: 'app=nginx'
      value: '2'
```
配置說明
- `triggers` 
  - `type`：指定為 `kubernetes-workload`，KEDA 會透過 `kube-apiserver`` 監控 Kubernetes 中 Pod 的數量。
  - `podSelector`：使用 LabelSelector 來選擇需要監控的 Pod（可使用 `,` 來指定多組條件）。
  - `value`：定義上游（nginx）與下游（redis）之間的關係。計算公式為：
    - 計算公式： `relation = (匹配的 Pod 數量) / (被擴展的 Workload Pod 數量)`。  

    比如，當 Nginx 的 Pod 數量擴展到 3 個時，算出的 relation 為 3 (大於配置的 2)，故會進行 Redis 的 Pod 的擴展。
      relation 計算範例：`3 = 3(nginx 副本數) / 1(redis 副本數)`。

部署 ScaledObject
```shell
kubectl apply -f keda-scaled-object.yml
```

## 模擬上游擴展 Pod
我們可以透過 `kubectl scale` 來模擬上游服務（nginx）因接收到更多流量而觸發 Pod 擴展。
```shell
kubectl scale deployment nginx --replicas 3
```
不久之後，Redis 也會從 1 個 Pod 擴展到 2 個 Pod：
```shell
kubectl get pod

NAME                     READY   STATUS              RESTARTS   AGE
nginx-7584b6f84c-5bqlm   1/1     Running             0          57m
nginx-7584b6f84c-bmhhv   1/1     Running             0          63m
nginx-7584b6f84c-lqzrx   1/1     Running             0          13s
redis-644585c74b-8v5rp   1/1     Running             0          63m
redis-644585c74b-wc2d4   0/1     ContainerCreating   0          2s
```

當 nginx 擴展到 5 個 Pod 時，Redis 也會相應擴展到 3 個 Pod：

```shell
kubectl scale deployment nginx --replicas 5

kubectl get pod

NAME                     READY   STATUS              RESTARTS   AGE
nginx-7584b6f84c-2gsrx   1/1     Running             0          17s
nginx-7584b6f84c-4nnx9   1/1     Running             0          17s
nginx-7584b6f84c-5bqlm   1/1     Running             0          59m
nginx-7584b6f84c-bmhhv   1/1     Running             0          65m
nginx-7584b6f84c-lqzrx   1/1     Running             0          118s
redis-644585c74b-8v5rp   1/1     Running             0          65m
redis-644585c74b-ct294   0/1     ContainerCreating   0          2s
redis-644585c74b-wc2d4   1/1     Running             0          107s
```

接著，我們模擬 nginx 的縮容，來觀察 Redis Pod 是否會同步縮減：

```shell
kubectl scale deployment nginx --replicas 2
```

由於大於 1 個 Pod 的擴縮容是由 [HPA] 處理的，為了避免頻繁變動，HPA 會參考過去 5 分鐘內的評估資料。因此，Redis Pod 的縮減可能會在 5 分鐘後才發生。
```shell
kubectl get pod

NAME                     READY   STATUS    RESTARTS   AGE
nginx-7584b6f84c-5bqlm   1/1     Running   0          73m
nginx-7584b6f84c-bmhhv   1/1     Running   0          79m
redis-644585c74b-8v5rp   1/1     Running   0          79m
```
> 📘 `ScaledObject`可以透過 advanced block 來調整 [HPA] 的行為，詳細內容可參閱 [KEDA 官方文件](https://keda.sh/docs/2.15/reference/scaledobject-spec/#overview)

# 小結
這三天我們透過 [KEDA] 實現了多種自動擴展策略，可以依據不同服務的需求，例如根據外部指標或上游服務的負載來調整 Pod 數量，提升系統的資源利用效率與服務穩定性。

明天我們將開始探討 Kubernetes 中管理網路流量的核心資源：`NetworkPolicy`，了解其如何有效控制 Pod 之間及外部流量的訪問。

# Refernce
- [KEDA]

[KEDA]: https://keda.sh/
[HPA]: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/
[HPA (Horizontal Pod Autoscaling)]: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/

[2023/day-29-Kubernetes 介紹-Pod 水平自動伸縮 (Horizontal Pod Autoscaler)]: https://ithelp.ithome.com.tw/articles/10336846

[kind]: https://kind.sigs.k8s.io/

[helm]: https://helm.sh/

[Kube-Prometheus]: https://prometheus-operator.dev/docs/getting-started/installation/