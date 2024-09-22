
# Day-17 Kustomize 介紹 - Patchs

# 前言
昨天已經使用 Kustomize 經典的目錄結構，並透過 Overlays 與 Patchs 對不同環境的需求，進行配置調整。

![https://github.com/YihongGao/picx-images-hosting/raw/master/20240914/截圖-2024-09-14-上午11.38.31.4918bfbffi.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240914/截圖-2024-09-14-上午11.38.31.4918bfbffi.webp)


今天會針對 Patchs 的使用方式做更詳細的介紹

# Patchs
Kustomize 支援兩種不同類型的 Patch 方式，
- JSON6902 Patch： 基於 [JSON6902 標準](https://datatracker.ietf.org/doc/html/rfc6902)
- Strategic Merge Patch：基於 Kubernetes 的 [Strategic Merge Patch](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-api-machinery/strategic-merge-patch.md)

直接看昨天使用的範例

> 代碼能參閱 [GitHub](https://github.com/YihongGao/iThome_30Day_2024/tree/main/resources/day17/kustomize-demo)

## JSON6902 Patch
```yaml
# kustomization.yaml

patches:
# inline JSON6902 Patch
# 修改名稱為 app-backen 與 product-backend 的 Deployment 的 cpu/memory 請求量
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
```
逐一解釋欄位用途
- `patch`: 定義 patch 內容的區塊，這裡透過 |- 使用 inline 直接定義 JSON6902 標準格式，不依賴其他檔案
  - `op: replace`：表示要替換某個欄位的值。
  - `path: /spec/template/spec/containers/0/resources/requests/cpu`： 要操作的欄位的在哪裡。
  - `value: "100m"`：該欄位的值改為什麼
- `target`： 要套用 patch 操作到哪些 Resource
  - `kind`: Resource kind
  - `name`: Resource name

當然不只有取代的功能能使用，還有 新增、複製、刪除等功能
``` yaml
# add: creates a new entry with a given value
- op: add
  path: /some/new/path
  value: value
# replace: replaces the value of the node with the new specified value
- op: replace
  path: /some/existing/path
  value: new value
# copy: copies the value specified in from to the destination path
- op: copy
  from: /some/existing/path
  path: /some/path
# move: moves the node specified in from to the destination path
- op: move
  from: /some/existing/path
  path: /some/existing/destination/path
# remove: delete's the node('s subtree)
- op: remove
  path: /some/path
# test: check if the specified node has the specified value, if the value differs it will throw an error
- op: test
  path: /some/path
  value: "my-node-value"
```

若是不希望 `kustomization.yaml` 檔案的行數過多，也能透過參照檔案的方式進行 JSON6902 patch
``` yaml
# kustomization.yaml
patches:
# JSON6902 Patch 
- patch:
  - path: patchs/lower-resource-requests-json6902.yml
    target:
      kind: Deployment
      name: app-backend|product-backend
```

```yaml
# patchs/lower-resource-requests-json6902.yml
- op: replace
  path: /spec/template/spec/containers/0/resources/requests/cpu
  value: "100m"
- op: replace 
  path: /spec/template/spec/containers/0/resources/requests/memory
  value: "300Mi"
```
透過 `path` 屬性參照到 JSON6902 格式的 JSON 或 YAML 檔，也能達成相同效果，並減少 `kustomization.yaml` 行數。

## Strategic Merge Patch
一樣先看範例
```yaml
# kustomization.yaml

patches:
# inline Strategic Merge Patch
# 修改 app-backen 的 cpu/memory 請求量
- patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: app-backen
      spec:
        template:
          spec:
            containers:
              - name: apps
                resources:
                  requests:
                    cpu: "100m"
                    memory: "300Mi"
# 修改 product-backend 的 cpu/memory 請求量
- patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: product-backend
      spec:
        template:
          spec:
            containers:
              - name: apps
                resources:
                  requests:
                    cpu: "100m"
                    memory: "300Mi"
```
相信讀者對 `patch` 中的內容一定很眼熟，內容就跟 Kubernetes 原生的 Deployment 格式一模一樣，只需要依照該 Resource 的格式，將要新增、取代的欄位寫在 patch 中即可，會自動依據 `apiVersion`、`kind`、`name` 進行調整。

另外 Strategic Merge Patch 一樣有能參照其他 YAML 檔案的方式


```yaml
# kustomization.yaml
patches:
- path: patchs/app-backend-low-resource-request.yml
- path: patchs/product-backend-low-resource-request.yml
```

```yaml
# patchs/app-backend-low-resource-request.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-backend
spec:
  template:
    spec:
      containers:
        - name: apps
          resources:
            requests:
              cpu: "100m"
              memory: "300Mi"
```

```yaml
# patchs/product-backend-low-resource-request.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-backend
spec:
  template:
    spec:
      containers:
        - name: apps
          resources:
            requests:
              cpu: "100m"
              memory: "300Mi"
```
與 inline Strategic Merge Patch 範例的寫法都能達到相同效果，但行數更短。

# 小結
我們能看出兩種 Patch 的方式跟風格都不相同
- JSON6902 Patch：命令式風格，需透過 `op` 屬性，精確的操作配置。
- Strategic Merge Patch：宣告式風格，需透過原生 YAML 格式撰寫出想調整的部分，由 Kustomize 自動進行新增/合併操作。

建議進行簡單的新增/修改 Patch 時，能優先考慮 `Strategic Merge Patch` 的方式，使用原生的 YAML 格式 與良好的 patch 檔名稱，能讓維護者很容易理解在做什麼操作。

當要操作 Array 屬性(如 env)、刪除、搬移時，能改用 `JSON6902 Patch` 來完成。

明天會介紹 Kustomize 的另一個功能：`Components`，這個用來跨 Overlays 的 Patch 功能。

# Refernce
- [Kustomize 官方文件 / patchs](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patches/#name-and-kind-changes)
- [christopher-adamson / Patches in Kustomize](https://www.linkedin.com/pulse/patches-kustomize-christopher-adamson-gaq4c)
- [openanalytics / kustomize-best-practices](https://www.openanalytics.eu/blog/2021/02/23/kustomize-best-practices/)