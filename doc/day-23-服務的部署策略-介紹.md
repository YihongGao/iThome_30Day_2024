# Day-23 服務的部署策略 - 介紹

# 前言
在軟體開發中，**唯一不變的就是改變**，如何在軟體更新迭代中降低服務中斷的風險，是一個重要的課題。

接下來幾天會介紹 服務的部署策略，並透過 Nginx、Argo Rollouts、Flipt 等工具來實現。
首先，讓我們了解幾種常見的部署策略
- 重建部署（Recreate）
- 滾動部署（Rolling Update）
- 藍綠部署（Blue/Green）
- 金絲雀部署（Canary）

> 以下介紹會將舊版本的服務稱為 **版本V1**，而新版本的服務稱為 **版本V2** 

# 重建部署（Recreate）
此策略會先關閉 **版本V1** 的服務，再啟動 **版本V2**，故會產生服務中斷。中斷時間長短取決於 **版本V1** 關閉到 **版本V2** 啟動之間所花費的時間。
![https://storage.googleapis.com/cdn.thenewstack.io/media/2017/11/c42fa239-recreate.gif](https://storage.googleapis.com/cdn.thenewstack.io/media/2017/11/c42fa239-recreate.gif)
圖檔來自: [Six Strategies for Application Deployment](https://thenewstack.io/deployment-strategies/)

顯而易見，這種方式不適合 Web 或 API 類服務，因為它會嚴重影響服務可用性。然而，對於排程類服務來說，這種策略非常合適，因為排程任務通常不希望多個 Pod 實例同時運行，以避免競爭條件（race condition）。

能透過設定 Kubernetres Deployment 的 `.spec.strategy.type==Recreate` 屬性來使用此策略。

# 滾動部署（Rolling Update）
滾動部署逐步切換流量到新版本服務上當 **版本V2** 的 Instance 建立好並準備好接收流量時，會將部分流量轉導至 **版本V2** 的 Instance，然後才關閉 **版本V1**，這個過程會反覆執行直到 **版本V2** 取代了 **版本V1** 全部的 Instance。
![https://storage.googleapis.com/cdn.thenewstack.io/media/2017/11/5bddc931-ramped.gif](https://storage.googleapis.com/cdn.thenewstack.io/media/2017/11/5bddc931-ramped.gif)
圖檔來自: [Six Strategies for Application Deployment](https://thenewstack.io/deployment-strategies/)


這種方式可以在不影響服務的情況下進行更新。Kubernetes Deployment 的默認部署策略就是滾動更新，並且可以根據服務需求來調整更新效率。

也能透過設定 Kubernetres Deployment 的 `.spec.strategy.type==RollingUpdate` 顯性指定此策略 並 搭配以下屬性控制更新效率
- **maxUnavailable**：更新過程中允許多少個 Pod 是不可用的，可以設置為絕對數字或百分比。
- **maxSurge**：更新過程中允許超過所需副本數的額外 Pod，這可以加速更新，值可以是絕對數字或百分比。

但 **Rolling Update** 仍有以下限制：
- **更新時間較長**： 逐步替換 Pod 會延長更新時間，特別是對大型應用程序。
- **新舊版本混合**： 更新期間，新舊版本同時存在，可能導致用戶體驗不一致，尤其在版本不兼容時。
- **無法精確控制流量**：難以在更新前驗證新版本服務是否正常。

# 藍綠部署（Blue/Green）
藍綠部署的原理是在不影響現有服務的情況下，先將 **版本V2** 完整部署並驗證服務正常後，再一次將流量從 **版本V1** 切換至 **版本V2**。
![https://storage.googleapis.com/cdn.thenewstack.io/media/2017/11/73a2824d-blue-green.gif](https://storage.googleapis.com/cdn.thenewstack.io/media/2017/11/73a2824d-blue-green.gif)
圖檔來自: [Six Strategies for Application Deployment](https://thenewstack.io/deployment-strategies/)

這個方式解決了**Rolling Update**的限制，能即時的切換服務版本，避免了版本混合問題，並且當 **版本V2** 部署完成後，能透過內部測試的方式提前驗證新版本是否符合要求。

而藍綠部署的代價是必須花費雙倍的系統資源，因為必須同時運行新舊版本的服務。
> 📘 Kubernetes workload 原生未提供此部署策略，需要搭配其他解決方案。

# 金絲雀部署（Canary）
金絲雀部署是一種逐步切換流量的策略，通常會將一部分（例如 10%）的流量轉移到 **版本V2**，其餘流量仍然發送到 **版本V1**。根據**版本V2** 的運行情況，逐步增加流量，直到完全替換 **版本V1**。

![https://storage.googleapis.com/cdn.thenewstack.io/media/2017/11/a6324354-canary.gif](https://storage.googleapis.com/cdn.thenewstack.io/media/2017/11/a6324354-canary.gif)
圖檔來自: [Six Strategies for Application Deployment](https://thenewstack.io/deployment-strategies/)

這個策略再系統進行高風險變更且對測試涵蓋率缺乏信心特別好用，它允許運維團隊逐步觀察新版本的運行狀況，並在發現問題時快速回滾。    
> 你永遠不知道尤其 Legacy 系統幫你準備了什麼驚喜。

![https://miro.medium.com/v2/resize:fit:1400/format:webp/1*tkXnZVtffjwzPC9zWNFBAg.jpeg](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*tkXnZVtffjwzPC9zWNFBAg.jpeg)
圖檔來自: [Bug Smashing — A Guide To Debug Your App](https://medium.com/mindorks/bug-smashing-a-guide-to-debug-your-app-11278d832e13)
> 📘 Kubernetes workload 原生未提供此部署策略，需要搭配其他解決方案。

# 小結
本文介紹了四種常見的服務部署策略：重建部署（Recreate）、滾動部署（Rolling Update）、藍綠部署（Blue/Green）、以及金絲雀部署（Canary）。每種策略都有其適用場景與優缺點。

- **重建部署（Recreate）**：簡單直接，但會產生服務中斷，不適合需要持續可用的服務。
- **滾動部署（Rolling Update）**：透過逐步替換來實現無中斷更新，但可能會遇到新舊版本混合的問題。
- **藍綠部署（Blue/Green）**：可即時切換新舊版本，避免版本混合，但需要雙倍資源來運行。
- **金絲雀部署（Canary）**：逐步轉移部分流量到新版本，適合高風險更新，但需要進一步的流量控制工具。

明天我們會透過 **Nginx Ingress Controller** 來實現 Kubernetes 原生支援的兩個部署策略：藍綠部署（Blue/Green）、以及金絲雀部署（Canary）。

# Refernce
- [Six Strategies for Application Deployment](https://thenewstack.io/deployment-strategies/)
- [Bug Smashing — A Guide To Debug Your App](https://medium.com/mindorks/bug-smashing-a-guide-to-debug-your-app-11278d832e13)
