
# Day-17 Kustomize 介紹 - Patches

# 前言
昨天，我們已經使用 Kustomize 的經典目錄結構，並透過 Overlays 和 Patches 來針對不同環境進行配置調整。

![https://github.com/YihongGao/picx-images-hosting/raw/master/20240914/截圖-2024-09-14-上午11.38.31.4918bfbffi.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240914/截圖-2024-09-14-上午11.38.31.4918bfbffi.webp)


今天我們將更詳細地介紹如何使用 Patches。

# Patches
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
### 欄位解釋：
- `patch`: 定義 patch 內容，此例使用 `|-` 來 inline 定義 JSON6902 標準格式，而不依賴其他檔案
  - `op: replace`：操作類型，此例使用 `replace` 表示要替換某個欄位的值。
  - `path: /spec/template/spec/containers/0/resources/requests/cpu`： 指定要操作的欄位位置。
  - `value: "100m"`：新的欄位值
- `target`： 要套用 patch 操作到哪些 Resource
  - `kind`: Resource kind
  - `name`: Resource name

除了替換 (replace)之外，JSON6902 還支援 **新增、複製、刪除** 等功能
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

### 使用外部檔案的方式
若是不希望 `kustomization.yaml` 檔案的行數過多，也能透過參照檔案的方式進行 JSON6902 patch
``` yaml
# kustomization.yaml
patches:
# JSON6902 Patch 
- patch:
  # 參照外部 JSON6902 Patch 檔案
  - path: patchs/lower-resource-requests-json6902.yml
    target:
      kind: Deployment
      name: app-backend|product-backend
```
外部檔案的內容如下：
```yaml
# patchs/lower-resource-requests-json6902.yml
- op: replace
  path: /spec/template/spec/containers/0/resources/requests/cpu
  value: "100m"
- op: replace 
  path: /spec/template/spec/containers/0/resources/requests/memory
  value: "300Mi"
```
透過 `path` 屬性參照到 JSON6902 格式的 JSON 或 YAML 檔，也能達成相同效果，並減少 `kustomization.yaml` 內容的行數。

## Strategic Merge Patch
我們直接看範例：
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
在 patch 中的內容與 Kubernetes 原生的 `Deployment` 配置幾乎相同。只需要依照該資源的格式，將需要新增或修改的欄位寫入 patch 中，Kustomize 會自動依據 `apiVersion`、`kind`、`name` 來進行合併與調整。

### 使用外部檔案的方式
Strategic Merge Patch 也支援參照外部檔案：
```yaml
# kustomization.yaml
patches:
- path: patchs/app-backend-low-resource-request.yml
- path: patchs/product-backend-low-resource-request.yml
```

外部檔案的內容如下：
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
這種方式與 inline 的寫法達到相同的效果，但能讓 kustomization.yaml 的行數更簡潔。

# 小結
兩種 Patch 的風格和操作方式各有不同：
- **JSON6902 Patch**：採用命令式風格，需使用 `op` 來精確操作配置，適合進行複雜的修改，如操作陣列、刪除或搬移。
- **Strategic Merge Patch**：採用宣告式風格，基於原生 YAML 格式撰寫，適合簡單的新增或修改操作，由 Kustomize 自動完成合併。

當進行簡單的新增或修改時，建議優先使用 `Strategic Merge Patch`，因為原生的 YAML 格式及清晰的 patch 檔案名稱，能讓維護者更容易理解操作內容。而對於更複雜的情況，如陣列屬性（如 env）、刪除或搬移操作，則可以使用 `JSON6902 Patch`。


明天，我們將介紹 Kustomize 的另一個功能：`Components`，這是用來跨 Overlays 的 Patch 功能。

# Refernce
- [Kustomize 官方文件 / patches](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patches/#name-and-kind-changes)
- [christopher-adamson / Patches in Kustomize](https://www.linkedin.com/pulse/patches-kustomize-christopher-adamson-gaq4c)
- [openanalytics / kustomize-best-practices](https://www.openanalytics.eu/blog/2021/02/23/kustomize-best-practices/)