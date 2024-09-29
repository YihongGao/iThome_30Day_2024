
# Day-27 服務的部署策略 - Argo Rollouts with Prometheus

# 前言
最後要來介紹 Argo Rollouts 中自動分析功能，自動分析新版本的服務是否符合預期，搭配**藍綠部署（Blue/Green）、金絲雀部署（Canary）** 全自動化更版的程序。

# 核心概念
- **AnalysisTemplate**：定義如何進行分析的模板，例如使用什麼 Metrics作為分析資料、分析頻率、成功/失敗閥值。

- **AnalysisRun**：運行分析任務的實例

如使用 金絲雀部署（Canary）的 Rollout 資源能引用 **AnalysisTemplate** 來自動分析新版本是否穩定，當新版本分析結果為通過時，能自動化擴大發佈的百分比，而失敗時，能自動退回穩定版本。
![https://miro.medium.com/v2/resize:fit:720/format:webp/0*IMRcBrRHh1_Ma4Oh
](https://miro.medium.com/v2/resize:fit:720/format:webp/0*IMRcBrRHh1_Ma4Oh
)
圖檔來源：[Minimize Impact in Kubernetes Using Argo Rollouts](https://medium.com/@arielsimhon/minimize-impact-in-kubernetes-using-argo-rollouts-992fb9519969)

## 環境準備
- **安裝 Prometheus**
參考 [Day11](https://ithelp.ithome.com.tw/articles/10348584) 的安裝方式

- **安裝 Ingress NGINX Controller**

  安裝方式參考： Day24 / 服務的部署策略 - Ingress NGINX Controller

- **安裝 Argo Rollouts**

  安裝方式參考：Day25 / 服務的部署策略 - Argo Rollouts

# 部署 Demo 服務
執行以下指令，將此 [Github repo](https://github.com/YihongGao/iThome_30Day_2024/blob/main/resources/day27/apps/canary/deploy.yml) 的 Demo 服務部署到 `ithome` namespeace
```shell
kubectl create namespace ithome
kubectl apply -f https://raw.githubusercontent.com/YihongGao/iThome_30Day_2024/refs/heads/main/resources/day26/apps/canary/deploy.yml
```

部署後會有以下資源：
- 1 個 **AnalysisTemplate**：`analysistemplates.argoproj.io/success-rate` 依據 **Prometheus** 中的 metrics 判斷新版本服務的 API 成功率
- 1 個 **Rollout**： `rollouts.argoproj.io/app-backend` 代表我們正在運行的服務。
- 2 個 **Service**： `service/app-backend-stable` 和`app-backend-preview`，分別對應穩定版本與新版本。
- 2 個 **Ingress**： 
  - `primary-ingress`：負責穩定版本的 Ingress
  - `app-backend-primary-ingress-canary`：由 Argo Rollouts 自動建立的 Ingress

## AnalysisTemplate
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
  - name: pod-name
  - name: canary-hash
  metrics:
  - name: success-rate
    initialDelay: 1m
    interval: 1m
    # NOTE: prometheus queries return results in the form of a vector.
    # So it is common to access the index 0 of the returned array to obtain the value
    successCondition: isNaN(result[0]) || result[0] >= 0.95
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus-k8s.monitoring.svc.cluster.local:9090
        query: |
          sum(rate(
            http_server_requests_seconds_count{status!~"5.*", pod=~"{{args.pod-name}}.*", rollouts_pod_template_hash="{{args.canary-hash}}"}[1m]
          )) /
          sum(rate(
            http_server_requests_seconds_count{pod=~"{{args.pod-name}}.*", rollouts_pod_template_hash="{{args.canary-hash}}"}[1m]
          ))
```
### 主要欄位介紹
- `args`: 定義分析模板的輸入參數，這些參數將在分析過程中被替換為實際的值。
- `metrics`: 定義分析的度量指標
    -  `initialDelay`: 1m: 分析開始前的延遲時間，這裡定義為 1 分鐘，允許系統在測試 Canary 部署之前有一段緩衝時間。
    -  `interval`: 1m: 分析每隔多久執行一次，這裡設定為每分鐘一次。
    - `successCondition: isNaN(result[0]) || result[0] >= 0.95`: 成功條件是一個布林表達式，表示當結果是 NaN 或成功率大於等於 95% 時，分析被認為成功。
        - `isNaN(result[0])`: 檢查結果是否為空或無法計算（例如 Prometheus 查詢結果無效）。
        - `result[0] >= 0.95`: 檢查第一個 Prometheus 查詢結果的值是否大於或等於 0.95（95% 的成功率）。
    - `failureLimit`: 3: 定義允許的失敗次數，當分析失敗次數達到 3 次時，會標記為失敗並停止分析，並觸發 Rollback。
- `provider`: 定義度量數據的提供者，這裡使用 Prometheus 作為數據源。
    - **`prometheus`**: Prometheus 是數據提供者的類型。
        - **`address`**: Prometheus 的 URL。
        - **`query`**: Prometheus 查詢語句，用來計算成功率。

簡單來說，這個 **AnalysisTemplate** 使用
Prometheus 中的 Metrics（http_server_requests_seconds_count）作為基準，當該值 >= 0.95（或無值）時，視為分析通過的條件。若該值不滿足此條件 3 次則視為失敗。

## Rollout
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  labels:
    app: app-backend
  name: app-backend
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
    # ... 省略
  strategy:
    canary:
      analysis:
        templates:
        - templateName: success-rate
        args:
        - name: pod-name
          value: app-backend
        - name: canary-hash
          valueFrom:
            podTemplateHashValue: Latest
      # ... 省略
      steps:
      - setWeight: 10
      - pause: {duration: 3m}
      - setWeight: 33
      - pause: {duration: 3m}
```
### 主要欄位介紹
- `analysis`：定義 **AnalysisTemplate** 配置
    - `templates`：引用哪些 **AnalysisTemplate** 的列表
        -  `templateName`：**AnalysisTemplate** 的名稱
    - `args`：傳入 **AnalysisTemplate** 的參數

於 strategy 中加上 `analysis` 區塊，來引用名稱為 `success-rate` 的 **AnalysisTemplate**，當進行金絲雀部署（Canary）時，若 `success-rate` 返回的分析結果為不通時，將會停止金絲雀部署（Canary），Rollback 到穩定版本。

# 驗證 金絲雀部署（Canary）+ AnalysisTemplate 的行為 - 分析通過
1. 模擬服務更新    
  使用 `kubectl edit` 更新 Rollout 資源的 Image Tag。
    ``` shell
    kubectl edit rollouts.argoproj.io/app-backend
    ```
    在編輯視窗中，將 spec.template.spec.containers[0].image 更新為：
    ```yaml
    spec:
      template:
        spec:
          containers:
          - image: luciferstut/app-backend-for-ithome2024:1.2
    ```
2. 透過 Argo Rollout Web UI 檢視分析狀態
轉發 UI 服務到本地 `http://localhost:3100`
    ```shell 
    kubectl argo rollouts dashboard
    ```
    用瀏覽器開啟 Dashboard（`http://localhost:3100`），能看到金絲雀發佈的進度，此例為開放 10% 流量。
    ![https://github.com/YihongGao/picx-images-hosting/raw/master/20240929/截圖-2024-09-29-下午11.54.02.7p9j77b9f.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240929/截圖-2024-09-29-下午11.54.02.7p9j77b9f.webp)
    點選下方 Analysis-<ramdon-id>，能檢視分析狀態
    ![https://github.com/YihongGao/picx-images-hosting/raw/master/20240929/截圖-2024-09-29-下午11.54.57.1ovekybg03.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240929/截圖-2024-09-29-下午11.54.57.1ovekybg03.webp)
    
    因為目前配置該 metrics 為空也視為成功，故不久後，會自動進入到下一個發佈階段（開放 10% 流量），最後自動進到全量開放，並關閉舊版本的 Pod。
    ![https://github.com/YihongGao/picx-images-hosting/raw/master/20240929/截圖-2024-09-30-上午12.00.50.5mns1mqvwb.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240929/截圖-2024-09-30-上午12.00.50.5mns1mqvwb.webp)

# 驗證 金絲雀部署（Canary）+ AnalysisTemplate 的行為 - 分析不通過
再來嘗試驗證分析不通過的使用案例
1. 模擬服務更新 
    使用 `kubectl edit` 更新 Rollout 資源的 Image Tag。
    ``` shell
    kubectl edit rollouts.argoproj.io/app-backend
    ```
    在編輯視窗中，將 spec.template.spec.containers[0].image 更新為：
    ```yaml
    spec:
        template:
        spec:
            containers:
            - image: luciferstut/app-backend-for-ithome2024:1.1
    ```
2. 模擬 API 失效
    使用 curl 向會 Response Http status 500 的 API 發出請求，產生 API 失敗的 Metrics 給 Prometheus 收集。
    ```shell
    kubectl exec -it <新版本的 Pod Name> -- bash -c 'for i in {1..50}; do curl http://localhost:8080/error-api; echo ""; done'
    ```
    從 Analysis 中能看到分析 Metrics 的結果為失敗(Failure)
    ![https://github.com/YihongGao/picx-images-hosting/raw/master/20240929/截圖-2024-09-30-上午12.24.54.4ckuvca1i2.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240929/截圖-2024-09-30-上午12.24.54.4ckuvca1i2.webp)

    當分析失敗的次數超過閥值時，會進入分析失敗的狀態：**Analysis failed**
    ![https://github.com/YihongGao/picx-images-hosting/raw/master/20240929/截圖-2024-09-30-上午12.29.04.6m3veturyx.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240929/截圖-2024-09-30-上午12.29.04.6m3veturyx.webp)
    
    並自動退回上一個穩定版本
    ![https://github.com/YihongGao/picx-images-hosting/raw/master/20240929/截圖-2024-09-30-上午12.29.19.6t73a9gxef.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240929/截圖-2024-09-30-上午12.29.19.6t73a9gxef.webp)

# 小結
今天介紹了如何結合 Argo Rollouts 和 Prometheus 進行自動化金絲雀部署。透過配置 **AnalysisTemplate**，我們可以自動監控新版本的關鍵指標，根據 API 成功率來判斷新版本的穩定性。在新版本表現良好時，系統會自動逐步增加流量；如果監測到異常，則會自動停止發佈並回滾到穩定版本。這不僅能有效降低服務中斷的風險，還能實現持續穩定的自動化發佈流程，大大提升了部署的安全性與效率。

**AnalysisTemplate** 也能使用其他資料來源來做分析，可參閱[官方文件](https://argoproj.github.io/argo-rollouts/features/analysis/)


# Refernce
- [Argo Rollout](https://argo-rollouts.readthedocs.io/en/stable/)
- [Minimize Impact in Kubernetes Using Argo Rollouts](https://medium.com/@arielsimhon/minimize-impact-in-kubernetes-using-argo-rollouts-992fb9519969)