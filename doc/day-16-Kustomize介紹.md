
# Day-16 Kustomize 介紹

# 前言
再 [2023 年的鐵人賽系列的分享](https://ithelp.ithome.com.tw/articles/10334732)中，我們實現了簡易版的 CI/CD pipeline，透過 Push-based 的方式實現 GitOps。
![old CI/CD](https://cdn.jsdelivr.net/gh/YihongGao/picx-images-hosting@master/20230913/架構圖.5holmyq61hg0.webp)

但其中仍有許多挑戰，例如
- 當有多個 Kubernetes 環境時，如何有效管理多組 YAML 文件？
- 當直接透過 `kubectl` 操作 Kubernetes Resource 時，環境與 Git 中的 manifest（YAML） 容易出現不一致的情況。

接下來的幾天，我們將逐步解決這些問題。今天，我們首先介紹 Kustomize，用來優化 YAML 的管理。

# 為什麼需要 Kustomize
在大多數產品或專案中，通常至少會有兩個環境（開發環境與生產環境），這些環境之間的配置大多相似，但會有一些細微差異，例如：
- **ConfigMap/Secrets 內容不同**：例如，開發環境與生產環境可能使用不同的資料庫連線字串或第三方服務的 URL。
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240914/截圖-2024-09-14-上午12.36.25.6t72neodmx.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240914/截圖-2024-09-14-上午12.36.25.6t72neodmx.webp)

- **資源配置不同**：開發環境通常使用較低的資源配置以節省成本，而生產環境則允許更高的資源請求與更多的 Pod 副本數量。
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240914/截圖-2024-09-14-上午12.36.38.2krvdky4uf.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240914/截圖-2024-09-14-上午12.36.38.2krvdky4uf.webp)

如果我們僅靠 copy/paste 來管理不同環境的 YAML 文件，最終很容易陷入混亂，難以追蹤每個環境之間的配置差異。   
Kustomize 允許使用者將資源進行模組化管理，並根據不同環境的需求應用額外的覆蓋設定（overlay），而不需要修改基礎配置，能有效的解決這個痛點。

# Kustomize 的基本概念
Kustomize 的核心概念包括以下幾個部分：

- **Base（基礎配置）**：    
Base 是一組可被多個環境共用的配置檔案，定義了資源的基本狀態。

- **Patches** ：     
Patches 是針對特定資源進行局部修改的配置主要方式。使用者可以用 JSON 或 YAML 格式來定義哪些部分的設定需要被覆蓋。

- **Overlay（覆蓋配置）**：   
Overlay 透過 `kustomization.yaml` 定義了每個環境的配置，包含要引用哪些 Base 與 要使用哪些 Patches，依照該環境的需求，將 Base 與 Patches 組合出該環境專屬的配置，無需複製所有 YAML 檔案。

- **Kustomization.yaml**：  
`kustomization.yaml` 檔案是 Kustomize 的核心，透過它能將多個資源（YAML）模組化與組裝。   

  每個 `kustomization.yaml` 能互相引用，例如 Overlay 的 `kustomization.yaml` 透過引用多個 Base 的 `kustomization.yaml` 組裝出該環境的完整配置。

簡單來說，Kustomize 將共用的 YAML 集中到 Base 層，減少了由 Copy-Paste 引起的維護問題。然後透過 Overlay 來區分不同環境，在 Overlay 中組裝 Base 與 套用 Patches 進行該環境的客製化調整，使多環境的配置維護變得更加輕鬆且直觀。

![https://github.com/YihongGao/picx-images-hosting/raw/master/20240914/截圖-2024-09-14-上午11.36.05.5c0xmb63wl.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240914/截圖-2024-09-14-上午11.36.05.5c0xmb63wl.webp)

每個環境最終部署的 YAML 配置將是 Base + 該環境的 Overlay（Patches）組合而成。
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240914/截圖-2024-09-14-上午11.38.31.4918bfbffi.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240914/截圖-2024-09-14-上午11.38.31.4918bfbffi.webp)


# Kustomize 的實際應用
我們拿上屆鐵人賽 demo 的服務來應用看看，共有兩個 deployment
- **app-backend**：一個 Spring boot application，依賴 `product-backend` 提供的 API。
- **product-backend**：一個 Spring boot application，提供 API 給 `app-backend` 調用

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
    └── production/ # 用於 生產環境 的 Overlay
        ├── configs
        │   ├── demo-config.yml
        |   └── kustomization.yml
        └── kustomization.yml # root kustomization.yml
```
### 結構說明：
- **Base（基礎配置）**： 
`kustomize-demo/bases` 目錄包含了 deployment、service、hpa 等所有環境共用的 YAML 檔案，作為共享配置。

- **Overlay（覆蓋配置）**： 
`kustomize-demo/overlays` 目錄包含針對每個環境的客製化配置，如 ConfigMap 和 Secrets，這些檔案在不同環境中配置內容通常會有所不同。develop 目錄為開發環境，production 目錄則為生產環境。

先來看 `bases/app-backend/kustomization.yml` 與 `bases/product-backend/kustomization.yml` 的內容
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yml
- service.yml
- hpa.yml
```
### 配置說明：
- `resources`: 定義要管理的資源列表（YAML 檔案）。當引用這個 `kustomization.yml` 時，Kustomize 會根據 `resources` 中列出的資源生成相應的 YAML 配置檔案。

再來看看使用於 生產環境 Overlay 的 `overlays/production/kustomization.yml` 
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
### 配置說明：
能看到跟 Base 的 kustomization.yml 很相似
- `resources`：這裡的 `resources` 不僅可以引用單個 YAML 檔案，還可以引用其他目錄中的 `kustomization.yml` 檔案。例如，這裡的 `../../bases/app-backend` 和 `../../bases/product-backend` 就是引用它們目錄下的 `kustomization.yml`，以將其管理的資源一併納入部署中。

- `namespace`：指定所有資源將被部署到 `ithome` namespace。

- `images`：用來替換指定的 container image Tag。在不同的環境中，這能方便地管理不同版本的 Container Image。例如，在這裡會將 `app-backend` 和 `product-backend` 的 Container Image替換為 1.0 版本。

簡單來說，當使用 `overlays/production/kustomization.yml` 進行部署時，Kustomize 將會部署以下資源：
- `overlays/production/configs` 目錄下 `kustomization.yml` 中管理的資源。
- `bases/app-backend` 目錄下 `kustomization.yml` 中管理的資源。
- `bases/product-backend` 目錄下 `kustomization.yml` 中管理的資源。
- 依據 `images.name` 與 `images.newTag` 更換資源中的 imageTag

讓我們透過 `kustomize` CLI 來檢視 YAML 內容。
```shell
# pwd
# current folder: /day16/kustomize-demo

kustomize build overlays/production > production.yml
```
> 📘 `kustomize` CLI 安裝方式參閱[官方文件](https://kubectl.docs.kubernetes.io/installation/kustomize/)

再來看看使用於 開發環境 Overlay 的 `overlays/develop/kustomization.yml` 

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
### 配置說明：
這個 Overlay 配置檔案除了定義基本的 `resources` 外，還使用了 `patches` 來對基礎配置進行客製化修改：
- `patches`：
  - 降低 `app-backend`、`product-backend` 這兩個 Deployment 的資源請求
  - 降低 `app-backend`、`product-backend` 這兩個 HPA 的副本數量 

- `namespace`：所有資源將會部署到 `ithome-dev` namespace 中，與生產環境（`ithome` namespace）區隔開來。
- `images`：替換 container image 的 Tag，開發環境使用 `latest` 版本的 Container Image，而非生產環境的 `1.0` 版本。這樣可以確保不同環境使用不同的容器映像版本，以適應不同的測試和發佈需求。

能透過 `kustomize` CLI 來檢視 YAML 內容。
```shell
# pwd
# current folder: /day16/kustomize-demo

kustomize build overlays/develop > develop.yml
```

最後我們實際把 Overlay 中 develop、production 的 kustomization 部署到 Kubernetes 中。
```shell
# kubectl create namespace ithome
kubectl apply -k overlays/production

# kubectl create namespace ithome-dev
kubectl apply -k overlays/develop
```

能用以下指令檢視資源是否都正確的依照 Overlay 的 `kustomization.yml` 進行對應的調整，並部署到指定的 namespace。 
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
今天介紹了 Kustomize 的核心概念，透過 Overlay 與 Patches 進行組裝與客製化，來滿足不同環境的配置需求，避免使用 Copy/Paste 配置的苦力與維運難題，讓管理 YAML 時更為容易且有效率。

明天會更詳細的說明 `Patchs` 的用法，讓讀者使用時能更得心應手。

# Refernce
- [Kustomize 官方文件](https://kubectl.docs.kubernetes.io/)
