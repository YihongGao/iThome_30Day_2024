
# Day-16 Kustomize 介紹

# 前言
再 [2023 年的鐵人賽系列的分享](https://ithelp.ithome.com.tw/articles/10334732)中，我們實現了簡易版的 CI/CD pipeline，透過 push base 的方式實現 GitOps。
![old CI/CD](https://cdn.jsdelivr.net/gh/YihongGao/picx-images-hosting@master/20230913/架構圖.5holmyq61hg0.webp)

但其中仍有許多挑戰，例如
- 當有多個 Kubernetes 環境時，如何管理 YAML
- 當直接透過 kubectl 操作 Kubernetes Resource 時，環境跟 manifest 就會不一致。

接下來幾天我們會陸續解決這些問題，首先先介紹的 Kustomize 來優化 yaml 的管理

# 為什麼需要 Kustomize
通常每個產品或專案都至少會有兩個環境(開發、生產環境)，不同環境之間時常有相似但略有不同的配置，例如
- ConfigMap/Secrets 內容不同：不同的資料庫連線字串 或 第三方服務的 URL
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240914/截圖-2024-09-14-上午12.36.25.6t72neodmx.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240914/截圖-2024-09-14-上午12.36.25.6t72neodmx.webp)

- 不同的資源配置：開發環境通常請求的資源配置較低，已節省成本，而生產環境允許較高的資源請求與副本量
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240914/截圖-2024-09-14-上午12.36.38.2krvdky4uf.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240914/截圖-2024-09-14-上午12.36.38.2krvdky4uf.webp)

這時若透過 copy/paste 來管理不同環境的 YAML，最終會是一團混亂，弄不清楚每個環境之間的差異。    
而 Kustomize 解決了這個問題，允許使用者將相同的資源模組化，並根據不同的環境需求，進行額外的設定覆蓋（overlay）而不改變基礎配置。

# Kustomize 的基本概念
Kustomize 的核心概念包括以下幾個部分：

- Base（基礎配置）:    
Base 是一組共享的配置檔案，這些檔案可以被多個環境共用。它們定義了資源的基本狀態，並且不會直接被修改。

- Overlay（覆蓋配置）:   
Overlay 是在 base 基礎上進行的客製化修改。針對不同的環境需求進行調整（如 開發、生產環境），Overlay 可以覆蓋 base 中的部分配置，而無需複製所有 YAML 檔案。

- Patches :     
Patches 是針對特定資源進行局部修改的配置主要方式。使用者可以用 JSON 或 YAML 格式來定義哪些部分的設定需要被覆蓋。

- Kustomization.yaml :  
每個目錄中的 kustomization.yaml 檔案是 Kustomize 的核心。這個檔案告訴 Kustomize 如何組合資源、進行修改和應用變更。它包括基礎配置、資源、patches 和其他配置選項。

簡單來說，Kustomize 將共用的 YAML 集中到 Base 層，減少 Copy Paste 衍生的維護問題，再透過 Overlay 作為區分環境的維度，於 Overlay 中進行 Patches，來調整每個環境之間的差異，更容易檢視環境之間的差異，讓多環境配置的維護更輕鬆。
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240914/截圖-2024-09-14-上午11.36.05.5c0xmb63wl.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240914/截圖-2024-09-14-上午11.36.05.5c0xmb63wl.webp)

每個環境最終會部署的 YAML 配置會是 Base + 該 Overlays(Patchs) 的配置
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240914/截圖-2024-09-14-上午11.38.31.4918bfbffi.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240914/截圖-2024-09-14-上午11.38.31.4918bfbffi.webp)


# Kustomize 的實際應用
我們拿上屆鐵人賽 demo 的服務來應用看看，共有兩個 deployment
- app-backend : 一個 Spring boot application，依賴 product-backend
- product-backend : 一個 Spring boot application，提供 API 給 app-backend 調用

將這兩個 Deployment 套上經典的 Kustomize 結構，完整檔案內容能參閱此 [Github Repo](https://github.com/YihongGao/iThome_30Day_2024/tree/main/resources/day16/kustomize-demo)
```shell
kustomize-demo/
├── bases/ # Base（基礎配置）
|   ├── app-backend/
|   |   ├── deployment.yml
│   |   ├── service.yml
│   |   ├── hpa.yml
|   |   └── kustomization.yml
|   ├── product-backend/
|   |   ├── deployment.yml
│   |   ├── service.yml
│   |   ├── hpa.yml
|   |   └── kustomization.yml
└── overlays/ # Overlay（覆蓋配置）
    ├── develop/ # 用於 開發環境 的 Overlay
    │   ├── configs
    |   │   ├── demo-config.yml
    |   |   └── kustomization.yml
    │   └── kustomization.yml # root kustomization.yml
    └── product/ # 用於 生產環境 的 Overlay
        ├── configs
        │   ├── demo-config.yml
        |   └── kustomization.yml
        └── kustomization.yml # root kustomization.yml
```
- Base（基礎配置）: 
`kustomize-demo/bases` 目錄作為 Base 目錄，配置了 deployment、service、hpa 這類每個環境都需要且大部分配置都相似的 YAML 檔，作為共享的配置檔案。

- Overlay（覆蓋配置）: 
`kustomize-demo/overlays` 目錄作為 Overlays 目錄，每個環境都有自己的目錄，其中通常包含 ConfigMap/Secrets 這類每個環境基本上都有，但配置內容差距較大的 YAML

先來看 `bases/app-backend/kustomization.yml` 與 `bases/product-backend/kustomization.yml` 的內容
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yml
- service.yml
- hpa.yml
```
- `resources`: 是定義要管理的資源列表(YAML)，當使用或引用此 `kustomization.yml` 時，會根據 `resources` 引用的資源產生出 YAML 配置檔。 

再來看看使用於 生產環境 Overlays 的 `overlays/production/kustomization.yml` 
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# 關聯到的 resource 都會部署到 ithome namespace
namespace: ithome

resources: 
- ./configs
- ../../bases/app-backend
- ../../bases/product-backend

images:
- name: luciferstut/app-backend-for-ithome2024:none
  newTag: "1.0"
- name: luciferstut/product-backend-for-ithome2024:none
  newTag: "1.0"
```
能看到跟 bases 的 kustomization.yml 很相似
- `resources`: 一樣是定義要管理的資源列表(YAML)，當指向其他目錄時，代表是將該目錄的 `kustomization.yml` 管理的資源都一起管理。

- `namespace`: 管理的 resource 會部署到指定的 namespace

- `images`: 將指定的 imageName 中的 Tag 進行替換，方便不同環境部署不同版本的 container Image。

簡單來說，當使用 `overlays/production/kustomization.yml` 進行部署時，會將以下資源部署到 Kubernetes 的 ithome namespace 中。
- `overlays/production/configs` 中 `kustomization.yml` 管理的資源
- `bases/app-backend` 中 `kustomization.yml` 管理的資源
- `bases/product-backend` 中 `kustomization.yml` 管理的資源
以上資源中使用的 imageName 若符合 `images.name` 時，會用 `images.newTag` 進行替換。

能透過 `kustomize` CLI 來檢視 YAML 內容。
```shell
# pwd
# current folder: /day16/kustomize-demo

kustomize build overlays/production > production.yml
```
> 📘 `kustomize` CLI 安裝方式參閱[官方文件](https://kubectl.docs.kubernetes.io/installation/kustomize/)

再來看看使用於 開發環境 Overlays 的 `overlays/develop/kustomization.yml` 

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# 關聯到的 resource 都會部署到 ithome-dev namespace
namespace: ithome-dev

resources: 
- ./configs
- ../../bases/app-backend
- ../../bases/product-backend

patches:
# used lower cpu/memory requests
- patch: |-
    - op: replace 
      path: /spec/template/spec/containers/0/resources/requests/cpu
      value: "100m"
    - op: replace 
      path: /spec/template/spec/containers/0/resources/requests/memory
      value: "300Mi"
  target:
    kind: Deployment
    name: app-backend|product-backend
# used lower maxReplicas config
- patch: |-
    - op: replace 
      path: /spec/minReplicas
      value: 1
    - op: replace 
      path: /spec/maxReplicas
      value: 2
  target:
    kind: HorizontalPodAutoscaler
    name: app-backend|product-backend

images:
- name: luciferstut/app-backend-for-ithome2024:none
  newTag: latest
- name: luciferstut/product-backend-for-ithome2024:none
  newTag: latest
```
能看到除了多個 `patches` 的區塊，用來調整 bases 中的 YAML 配置，依此案例就是用來減少開發環境成本。
- 降低 `app-backend`、`product-backend` 這兩個 Deployment 的資源請求
- 降低 `app-backend`、`product-backend` 這兩個 HPA 的副本數量 

另外透過 `namespace`、`images` 提供使用不同 image 或 namespace 的能力。

能透過 `kustomize` CLI 來檢視 YAML 內容。
```shell
# pwd
# current folder: /day16/kustomize-demo

kustomize build overlays/develop > develop.yml
```

最後我們實際把 overlays 中 develop、production 的 kustomization 部署到 Kubernetes 中。
```shell
# kubectl create namespace ithome
kubectl apply -k overlays/production

# kubectl create namespace ithome-dev
kubectl apply -k overlays/develop
```

能用以下指令檢視資源是否都正確的依照 overlays 的 `kustomization.yml` 進行對應的調整，並部署到指定的 namespace。 
```yaml
# HPA replicas 應為 1 to 2
kubectl get deployment,svc,hpa,cm -n ithome

# resource.requests 與 base 相同
# 使用指定的 image tag
kubectl describe deployments.apps app-backend -n ithome
kubectl describe deployments.apps product-backend -n ithome

# HPA replicas 應為 1 to 2
kubectl get deployment,svc,hpa,cm -n ithome-dev

# resource.requests 應被降低
# 使用 latest image tag
kubectl describe deployments.apps app-backend -n ithome-dev
kubectl describe deployments.apps product-backend -n ithome-dev
``` 

# 小結
今天初步應用了 Kustomize 的幾個核心概念，能依照不同環境的需求對配置進行調整，且避免 Copy/Paste，讓管理 YAML 時更為容易。

明天會更詳細的說明 `Patchs` 的用法，讓讀者使用時能更得心應手。

# Refernce