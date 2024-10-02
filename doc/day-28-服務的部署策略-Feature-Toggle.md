# Day-28 服務的部署策略 - Feature Toggle

# 前言
我們介紹了透過 **Ingress NGINX Controller、Argo Rollouts** 實現 Pod 維度的部署策略，但若要長時間運行運行新舊版本的 Pod，需佔用多一倍的運算資源。

另外再微服務架構中，也會面對一些挑戰，例如：A服務與B服務有 API 依賴性，當 A 服務走新版本邏輯程時，必須與 B 服務中的新版本邏輯，反之都需要走舊版本邏輯。

這時候我們能透過 **Feature Toggle**，實現代碼維度的部署策略，達成靈活、快速地發布新功能且盡可能的降低風險。

# 什麼是 Feature Toggle？
**Feature Toggle** 是一種設計模式，讓開發團隊可以在不影響系統的情況下，靈活地控制新功能的開啟或關閉，在功能完成前不對使用者開放。當功能準備就緒時，開發團隊只需開啟開關即可立即發布新功能，而無需重新部署。


這是一個最簡單的 **Feature Toggle** 代碼範例
```kotlin
fun doSomething(){
    // 當 Feature Toggle 為 True 時，使用新邏輯，反之使用舊邏輯
    if( featureIsEnabled("use-new-feature") ){
        return doSomethingWithNewLogic(); 
    }else{
        return doSomethingWithOldLogic(); 
    }
}
```
此範例中，`featureIsEnabled(String)` 這個函數作為 **Toggle Router**，負責管理功能的啟用/關閉，而 **Toggle Router** 的實現方式將直接影響 **Feature Toggle** 是否好用與可靠。

今天，將使用 [Flipt] 作為 **Toggle Router** 的實現，來介紹 **Feature Toggle** 的使用方式。

# Flipt
Flipt 是一個 Open soruce 的 Feature Toggle 平台，提供以下功能
- **靈活的規則引擎**：根據條件啟用功能，支持金絲雀發布（Canary）和其他複雜的條件邏輯。
- **高效能**：Flipt 提供低延遲的 API，並支持多種數據存儲，包括 Redis，確保高性能。
- **動態調整**：無需重新部署服務，開發團隊可以隨時調整功能的啟用規則。
- **Web UI**：有 UI 就給讚。
- **豐富的 SDK**：支援多種主流語言(Java、Go、Python..等)

## 核心概念
- **Flags**：**Flags**Flipt 中的功能開關，用於控制應用程式中功能的啟用與禁用。分為兩種類型：
    - **Boolean**：返回 `true` 或 `false`，適用於簡單的功能開關。
    - **Variant**：返回不同的變體值（如 `dark`、`light`、`auto`），適用於多種狀態的功能，例如調整 UI 的主題色。
     
- **Segments**：根據特定條件（如地區或用戶類型）對用戶進行分群，以便對不同群體應用不同的功能標籤。

- **Rules / Rollouts**：定義哪些用戶群（Segments）符合某 Flag 的條件，並根據條件或百分比分發不同的結果，例如，30% 的用戶使用 `dark` 色系。

- **Evaluation**：Flipt 會根據 Evaluation 請求來決定是否啟用某個 Flag，Evaluation 請求中包含以下資訊。
    - `Entity Id`：通常是能識別用戶的唯一值，如 userId、email，`Entity Id` 將作為百分比發佈演算法計算的參數。
    - `Context`：通常用來攜帶與用戶身份無關，提供資訊來識別用戶屬於什麼 **Segments**，如：用戶地區資訊。

## 使用案例

### Beta 測試
生產環境部署時，僅將新功能僅開放給屬於內部人員的帳號，當內部人員驗證新功能正確時，再開放給外部使用者。
![https://miro.medium.com/v2/resize:fit:720/format:webp/1*3Jf_TpC-_KqMqBKYWD6yRQ.jpeg](https://miro.medium.com/v2/resize:fit:720/format:webp/1*3Jf_TpC-_KqMqBKYWD6yRQ.jpeg)
圖檔來自 [Integration of both Canary Deployment and feature flagging design patterns simplified]

### 金絲雀發佈(Canary)
典型的金絲雀部署，由小量開放新版本功能，確認功能穩定在逐漸放大比例
![https://github.com/YihongGao/picx-images-hosting/raw/master/20241002/截圖-2024-10-02-下午9.32.25.73tx7ivvde.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20241002/截圖-2024-10-02-下午9.32.25.73tx7ivvde.webp)
圖檔來自 [Feature Toggle Makes Development Faster and Safer @ TECHPULSE 2023]

### A/B Tesing
類似金絲雀部署，但通常是為了商業目的，例如：分析新/舊版本的訂單轉換率較高 或 用戶停留時間更長。
![https://configcat.com/blog/assets/images/half-users-6beaf737e668c20287489f069c64efd3.jpeg](https://configcat.com/blog/assets/images/half-users-6beaf737e668c20287489f069c64efd3.jpeg)
圖檔來自 [The role of feature flags in A/B testing]

### 架構圖 
![https://mintlify.s3-us-west-1.amazonaws.com/flipt/images/architecture/flipt_dark.svg](https://mintlify.s3-us-west-1.amazonaws.com/flipt/images/architecture/flipt_dark.svg)

透過 **Flipt**，我們就不需要部署多個版本的 Deployment，只需要在應用程序中向 Flipt server 發出 **Evaluation** 請求，應用程序依照 **Evaluation** 的回應執行對應的代碼。

當有多組服務要同時應用同一個版本時，只要使用同樣的資訊向 **Flipt** 發出 **Evaluation** 請求都能得到相同的回應值 (假設 **Segments**、**Rules / Rollouts** 都未改變)，能完美解決服務依賴的版本控制難題。

## 安裝方式
能使用 Helm 輕鬆的安裝到 Kubernetes 中
```shell
#  add the Flipt Helm repository
helm repo add flipt https://helm.flipt.io

# create namespace
kubectl create ns flipt

# install
helm install flipt flipt/flipt --namespace=flipt
```
能看到 namespace：`flipt` 中有 flipt 的 Pod 在運行。
```shell
kubectl get pod -n flipt

# Output
NAME                     READY   STATUS    RESTARTS   AGE
flipt-6c487dc75f-h9xg8   1/1     Running   0          3m57s
```

## 配置 Feature toggle
我們來配置一個依據 用戶地區 決定功能開關的 Feature toggle

### 1. 於本地開啟 Flipt UI
將 flipt UI 轉發到 `http://localhost:8080`，方便配置
```shell
kubectl port-forward services/flipt 8080:8080
```
用瀏覽器開啟 `http://localhost:8080`
![https://github.com/YihongGao/picx-images-hosting/raw/master/20241002/截圖-2024-10-02-下午11.46.54.6f0nnn14l0.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20241002/截圖-2024-10-02-下午11.46.54.6f0nnn14l0.webp)

### 2. 建立 **Segments**
1. 點選 **Segments** 頁籤，並點選 **Create Segment** 按鈕    
1. 填寫以下資訊後，點選 **Create**
    - Name：`Asia-User`
    - Key：`Asia-User`
1. 點選下方**New Constraint**，設定以下配置，並點選 **Create** 
    - Type：`String`
    - Property：`region`
    - Operator：`==`
    - Value：`Asia`

![https://github.com/YihongGao/picx-images-hosting/raw/master/20241002/截圖-2024-10-03-上午12.12.19.6wqpc8z69z.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20241002/截圖-2024-10-03-上午12.12.19.6wqpc8z69z.webp)

### 3. 建立 Boolean Flag 並定義 Rollout
1. 點選 **Flags** 頁籤，並點選 **Create Flag** 按鈕    
1. 填寫以下資訊後，點選 **Create**
    - Name：`Demo-Feature-Toggle`
    - Key：`Demo-Feature-Toggle`
    - Type：`Boolean`
1. 點選下方**New Rollout**，設定以下配置，並點選 **Create** 
    - Type：`Segment`
    - Segment：`Asia-User`
    - Value：`True`
![https://github.com/YihongGao/picx-images-hosting/raw/master/20241002/截圖-2024-10-03-上午12.04.31.3k7zhv8zam.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20241002/截圖-2024-10-03-上午12.04.31.3k7zhv8zam.webp)

## 驗證 Feature toggle 
能透過 Flipt UI 直接測試 Flag
### 透過 UI 驗證
1. 點選 **Console** 頁籤，並填寫以下值
    - Flag Key：選 `Demo-Feature-Toggle`
    - Request Context：`{"region": "Asia"}`
2. 點選 **Evaluate** 按鈕觸發 **Evaluation** 請求，右邊區塊能看到 Response 內容
```json
{
  "enabled": true,
  "reason": "MATCH_EVALUATION_REASON",
  "requestId": "fbad0ce1-447f-4ef0-af46-73bab293ed5b",
  "requestDurationMillis": 2.39275,
  "timestamp": "2024-10-02T16:16:40.784382764Z",
  "flagKey": "Demo-Feature-Toggle"
}
```
其中 `enabled` 欄位代表了 Boolean Flag 中的配置
![https://github.com/YihongGao/picx-images-hosting/raw/master/20241002/截圖-2024-10-03-上午12.17.31.6f0nno58hd.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20241002/截圖-2024-10-03-上午12.17.31.6f0nno58hd.webp)
反之，`region` 為空或其他值時，`enabled` 欄位會返回 `false`
![https://github.com/YihongGao/picx-images-hosting/raw/master/20241002/截圖-2024-10-03-上午12.17.35.8l029fww8h.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20241002/截圖-2024-10-03-上午12.17.35.8l029fww8h.webp)

## 使用 SDK 串接 Flipt

```java
import io.flipt.api.FliptClient;
import io.flipt.api.evaluation.models.*;

public class Main {
  public static void main(String[] args) {
    FliptClient fliptClient = FliptClient.builder().url("YOU_FLIPT_SERVER_URL").build();
    Map<String, String> context = new HashMap<>();

    context.put("region", "Asia");

    EvaluationRequest variantEvaluationRequest =
        EvaluationRequest.builder()
            .namespaceKey("default")
            .flagKey("Demo-Feature-Toggle")
            .entityId("userId")
            .context(context)
            .build();
    
    BooleanEvaluationResponse booleanEvaluationResponse = fliptClient.evaluation().evaluateBoolean(variantEvaluationRequest);
        
    // 依據 Flipt Evaluation Response 值判斷使用新/舊版邏輯
    if(booleanEvaluationResponse.isEnabled()){
        return doSomethingWithNewLogic();
    }else{
        return doSomethingWithOldLogic();
    }
  }
}
```
只要應用程序將實際的用戶地區填入 `context` 中，使用 **Evaluation** 時，Flipt 就會依據該 **Flag** 的 **Segments** 與 **Rules / Rollouts** 運算出對應的值，讓應用程序使用。

對 Canary 發布有興趣的讀者，能自行嘗試配置 **Rollouts** 的 Type 為 `Threshold` 或調整其他條件看看。

# 小結
今天介紹了如何利用 **Feature Toggle** 結合 [Flipt] 來靈活管理功能發布。相比傳統的金絲雀部署，Feature Toggle 專注於代碼層面的功能控制，讓開發團隊能快速啟用或禁用功能，無需重新部署。Flipt 提供高效的 API、簡單的 UI 介面，以及多種語言 SDK，讓配置更加靈活。無論是 Beta 測試、金絲雀發布，還是 A/B 測試，Flipt 都能幫助開發團隊更好地管理和驗證新功能的運行，提升部署效率與穩定性。

# Refernce
- [Flipt]
- [Integration of both Canary Deployment and feature flagging design patterns simplified]
- [Feature Toggle Makes Development Faster and Safer @ TECHPULSE 2023]
- [The role of feature flags in A/B testing]

[Flipt]: https://www.flipt.io/

[Integration of both Canary Deployment and feature flagging design patterns simplified]: https://medium.com/@ar.aldhafeeri11/simple-expressjs-canary-deployment-using-feature-flagging-design-pattern-8fc08f277851

[Feature Toggle Makes Development Faster and Safer @ TECHPULSE 2023]: https://speakerdeck.com/line_developers_tw/feature-toggle-makes-development-faster-and-safer-at-techpulse-2023?slide=3

[The role of feature flags in A/B testing]: https://configcat.com/blog/2022/05/02/what-is-ab-testing/