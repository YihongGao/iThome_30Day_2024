### Pod 行為
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240909/截圖-2024-09-09-下午4.44.02.8vmv59zo4b.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240909/截圖-2024-09-09-下午4.44.02.8vmv59zo4b.webp)

- frontend Pod：
  模擬前端服務，監聽 80 port 的 http 請求，並將請求轉發給 backend 8080 port 處理。

- backend Pod：
  模擬後端服務，監聽 8080 port，收到請求後，調用 db 的 6379 port 取得資料，並把資料回傳給前端。

- db Pod：
  模擬資料庫，監聽 6379 port 並提供資料。
