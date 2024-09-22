
# Day-12 KEDA 介紹 - 基於 Message Queue 自動擴展 Pod

# 前言
昨天我們介紹了如何根據 Prometheus 中的 metrics 來自動擴展 Pod。今天，我們將使用 GCP 的訊息佇列服務 **Pub/Sub**，並實現當佇列中沒有訊息需要處理時，將 Pod 縮減至 0，以節省運算資源。

# 環境準備
在 Kubernetes 中需要進行以下準備工作：

1. 配置 GCP Pub/Sub 的 Topic 和 Subscription。
1. 配置 GCP Service Account 並產生 `gcp-service-account-credential.json` 憑證文件。
1. 將 `gcp-service-account-credential.json` 透過 Kubernetes Secrets 儲存至 Cluster。
1. 部署相關 Workload。

## 配置 GCP Pub/Sub topic & subscription
透過 UI 或 `gcloud` command line tool 進行配置
```shell
# 需要先 login 與 指定 project
# gcloud auth login 
# gcloud config set project PROJECT_ID

# 建立 topic
TOPIC_NAME=ithome2024.day11.topic
gcloud pubsub topics create $TOPIC_NAME

SUBSCRIPTION=ithome2024.day11.subscription
# 建立 subscription
gcloud pubsub subscriptions create --topic $TOPIC_NAME $SUBSCRIPTION
```

![https://github.com/YihongGao/picx-images-hosting/raw/master/20240903/截圖-2024-09-03-下午11.47.24.26lf8djyue.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240903/截圖-2024-09-03-下午11.47.24.26lf8djyue.webp)

## 配置 GCP Service Account 與 產生 gcp-service-account-credential.json
透過 UI 或 `gcloud` command line tool 進行配置
```shell
PROJECT_ID=${you GCP project_id}
SERVICE_ACCOUNT_NAME=ithome-demo

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:$SERVICE_ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/pubsub.subscriber"

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:$SERVICE_ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/pubsub.viewer"

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:$SERVICE_ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/monitoring.viewer"

# Download service account key
# ⚠️注意：這個 key 不能外流，否則有 GCP 被外人操控的風險
gcloud iam service-accounts keys create ./gcp-service-account-credential.json \                                                       
    --iam-account=$SERVICE_ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com
```

![https://github.com/YihongGao/picx-images-hosting/raw/master/20240903/截圖-2024-09-03-下午11.46.56.1zi7cxx3jc.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240903/截圖-2024-09-03-下午11.46.56.1zi7cxx3jc.webp)

## 將 gcp-service-account-credential.json 透過 Kubernetes secrets 儲存到 Kubernetes cluster 內
透過 kubectl 建立 secret
```shell
kubectl create secret generic pubsub-secret --from-file=gcp-service-account-credential.json=./gcp-service-account-credential.json
```
檢查是否成功建立
```shell
kubectl get secrets 

NAME            TYPE     DATA   AGE
pubsub-secret   Opaque   1      110m
```

## 部署 workload
> 代碼能參閱 [GitHub](https://github.com/YihongGao/iThome_30Day_2024/tree/main/resources/day12)

我們部署一個 listen GCP Pub/Sub 的 spring boot Application
```yaml
# keda-pubsub-demo.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: keda-pubsub-demo
  name: keda-pubsub-demo
  namespace: ithome
spec:
  replicas: 1
  selector:
    matchLabels:
      app: keda-pubsub-demo
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: keda-pubsub-demo
        scrape: spring-boot-prometheus-exporter
    spec:
      volumes:
        - name: gcp-sa-credentials
          secret:
            secretName: pubsub-secret
            items:
            - key: gcp-service-account-credential.json
              path: gcp-service-account-credential.json
      containers:
      - image: luciferstut/spring-boot-application-for-ithome2024:day12
        imagePullPolicy: Always
        name: spring-boot-application-for-ithome2024
        env:
        - name: GOOGLE_CLOUD_PROJECT
          value: yihonggao-1548227860432
        - name: PUBSUB_ITHOME2024_DAY11_SUBSCRIPTION
          value: projects/yihonggao-1548227860432/subscriptions/ithome2024.day11.subscription
        - name: GOOGLE_APPLICATION_CREDENTIALS
          value: /etc/gcp/gcp-service-account-credential.json
        - name: GOOGLE_APPLICATION_CREDENTIALS_JSON
          valueFrom:
            secretKeyRef:
              name: pubsub-secret
              key: gcp-service-account-credential.json
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
        startupProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          failureThreshold: 18
          periodSeconds: 5
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
        volumeMounts:
        - name: gcp-sa-credentials
          readOnly: true
          mountPath: "/etc/gcp/"
```

部署該 Deployment ，並嘗試 Push 一個 message 看是否能成功消化。
```shell
# 推送 message to Pub/Sub Topic
gcloud pubsub topics publish $TOPIC_NAME --message='Hello, Pub/Sub!'

# 檢視 Pod log
kubectl logs -f $YOU_POD_NAME
```

看到這一行時，代表 Spring boot Application 成功與 GCP Pub/Sub 連上並能處理 message 了。
```log
2024-09-03T16:12:26.052Z  INFO 1 --- [iThome-2024] [sub-subscriber2] c.e.iThome_2024.config.PubSubConfig      : Message arrived via an inbound channel adapter from sub-one! Payload: Hello, Pub/Sub!
```

接下來要來配置 KEDA 來自動擴展此 Deployment

# 配置 KEDA

## 配置 TriggerAuthentication
為了讓 KEDA 能與 GCP 正常連線，我們需要配置 **TriggerAuthentication**，使 KEDA 知道如何攜帶憑證來進行認證與授權檢查。

```yaml
# keda-scaled-auth.yml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-trigger-auth-gcp-credentials
spec:
  secretTargetRef:
  - parameter: GoogleApplicationCredentials
    name: pubsub-secret
    key: gcp-service-account-credential.json
```
重點欄位解釋：
- `secretTargetRef`：從 Secrets 取得憑證
    - `parameter`：憑證類型，`GoogleApplicationCredentials` 代表 GCP Service Account 的 json 憑證.
    - `name`：指定從哪個 Secrets 中取得憑證。
    - `key`：Secrets  中存放憑證的欄位名稱。

稍後，我們將在 ScaledObject 配置中使用到這個 **TriggerAuthentication** 物件。

```shell
kubectl apply -f keda-scaled-auth.yml
```

## 配置 ScaledObject
```yaml
# keda-scaled-object.yml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: keda-pubsub-demo
  namespace: ithome
spec:
  fallback:
    failureThreshold: 3
    replicas: 1
  pollingInterval: 5
  cooldownPeriod: 60
  idleReplicaCount: 0
  minReplicaCount: 1
  maxReplicaCount: 3
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: keda-pubsub-demo
  triggers:
  - type: gcp-pubsub
    authenticationRef:
      name: keda-trigger-auth-gcp-credentials
    metadata:
      mode: "SubscriptionSize"
      value: "10"
      activationValue: "5"
      subscriptionNameFromEnv: PUBSUB_ITHOME2024_DAY11_SUBSCRIPTION
```
可以看到新增了許多欄位，首先從 `triggers` 開始說明：
- `type`: 設置為 `gcp-pubsub`，代表監聽目標為 GCP Pub/Sub 服務
    - `authenticationRef.name`: 使用上一步建立的 `TriggerAuthentication` 做為與 GCP 溝通的憑證
    - `mode`, `value`: 這這兩個屬性組合表示，當 Pub/Sub 訂閱中的未處理訊息數超過 10 筆時，KEDA 將觸發擴展操作，增加 Pod 副本數。
    - `idleReplicaCount`, `activationValue`: 這兩個屬性組合表示，當未處理訊息數低於 5 筆時，系統將進入 `Inactive` 狀態，並將 Pod 副本數縮減為 0。反之，進入 `Active` 狀態後，HPA 會接手擴容，並確保 Pod 副本數介於 `minReplicaCount` 和 `maxReplicaCount` 之間。
        > 有關 `Active` 與 `Inactive` 狀態，更多資訊，能參閱此[官方文件](https://keda.sh/docs/2.15/concepts/scaling-deployments/#activating-and-scaling-thresholds)
    - `cooldownPeriod`: 當未處理訊息數持續低於 5 筆超過 60 秒後，才會進入 `Inactive` 狀態。

其他欄位在之前的介紹中已有說明，不再重複。更詳細的介紹請參閱 [官方文件](https://keda.sh/docs/2.15/reference/scaledobject-spec/) 

這個 ScaledObject 利用 GCP 的監控指標來檢查 Pub/Sub 訂閱的佇列長度，當佇列長度持續低於 5 筆超過 60 秒時，將 Pod 副本數縮減為 0，以節省資源；當佇列長度超過 5 筆時，則啟動 Pod 處理訊息。若佇列長度超過設定的值，系統會擴展更多的 Pod 來協同處理訊息。

配置 ScaledObject
```shell
kubectl apply -f keda-scaled-object.yml
```
過段時間應該能看到，以下資訊
```shell
kubectl get scaledobjects.keda.sh 

NAME               SCALETARGETKIND      SCALETARGETNAME    MIN   MAX   TRIGGERS     AUTHENTICATION                      READY   ACTIVE   FALLBACK   PAUSED    AGE
keda-pubsub-demo   apps/v1.Deployment   keda-pubsub-demo   1     3     gcp-pubsub   keda-trigger-auth-gcp-credentials   True    False    False      Unknown   9m50s
```

此 demo 中
- `READY` 欄位: 代表是否成功連上 GCP
- `ACTIVE` 欄位: 代表 queue 長度是否超過 `activationValue` 配置

能看到 `ACTIVE` 為 False 時，Deployment 的 Pod 會歸 0
```shell
kubectl get deployments.apps 

NAME               READY   UP-TO-DATE   AVAILABLE   AGE
keda-pubsub-demo   0/0     0            0           162m

kubectl get pod
No resources found in ithome namespace.
```

讓我們打些 message 到 GCP Pub/Sub 試試看
```shell
for i in {1..40}; do gcloud pubsub topics publish $TOPIC_NAME --message='Hello, Pub/Sub'; done
```
> ⚠️ 因為 GCP monitoring 採樣數據頻率較慢，需要稍等一下

能看到 `ScaledObject` ACTIVE 改為 `True` 並啟動 Pod 了
```shell
kubectl get scaledobjects.keda.sh
NAME               SCALETARGETKIND      SCALETARGETNAME    MIN   MAX   TRIGGERS     AUTHENTICATION                      READY   ACTIVE   FALLBACK   PAUSED    AGE
keda-pubsub-demo   apps/v1.Deployment   keda-pubsub-demo   1     3     gcp-pubsub   keda-trigger-auth-gcp-credentials   True    True     False      Unknown   19m

kubectl get pod
NAME                                READY   STATUS    RESTARTS   AGE
keda-pubsub-demo-7f8944cf7f-9qfxb   1/1     Running   0          46s
keda-pubsub-demo-7f8944cf7f-h6br5   0/1     Running   0          31s
keda-pubsub-demo-7f8944cf7f-rphh5   0/1     Running   0          31s
```

# 環境清理
因 `ScaledObject` 監聽 `Pub/Sub` 的方式是透過 [GCP Monitoring](https://cloud.google.com/monitoring) 來感知 Queue 長度，所以驗證完成後，記得把 `ScaledObject` 移除，避免產生 GCP 費用。
```shell
kubectl delete scaledobjects.keda.sh keda-pubsub-demo
```

# 小結
今天我們介紹了如何使用 KEDA 根據 Message Queue 的長度來進行自動擴/縮容。當 message consumer 資源消耗大且訊息頻率不穩定的情況下，這種策略能有效避免資源長時間被佔用和浪費。

明天會介紹本系列最後一個 KEDA 使用案例，若對更多使用案例有興趣，能參閱 [官方文件/scalers](https://keda.sh/docs/2.15/scalers/) 中許多能作為事件來源的選擇。

# Refernce
- [KEDA]

[KEDA]: https://keda.sh/
[HPA]: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/
[HPA (Horizontal Pod Autoscaling)]: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/

[2023/day-29-Kubernetes 介紹-Pod 水平自動伸縮 (Horizontal Pod Autoscaler)]: https://ithelp.ithome.com.tw/articles/10336846

[kind]: https://kind.sigs.k8s.io/

[helm]: https://helm.sh/

[Kube-Prometheus]: https://prometheus-operator.dev/docs/getting-started/installation/