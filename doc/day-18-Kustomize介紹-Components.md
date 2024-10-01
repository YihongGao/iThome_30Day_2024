# Day-18 Kustomize 介紹 - Components

# 前言
昨天介紹了如何在單一個 Overlay 的 `Kustomization.yaml` 中透過 Patch 調整 YAML。

今天要介紹的 **Components** 是 Kustomize 中一個更高級的概念，允許將 Resource 的片段進行 **模組化**，並讓多個 Overlays 共享，從而提升重用性與維護性。

## 為什麼要使用 Components？
實務上的環境管理可能有更複雜的情境出現，例如
- 開發環境：較少的安全限制，較鬆散的 Rolling 策略
- 預生產環境：完整的安全限制，較鬆散的 Rolling 策略
- 生產環境：完整的安全限制，嚴謹的 Rolling 策略

這時將 安全限制 與 Rolling 策略的配置，透過 Componentes 進行模組化，並於指定的環境使用該配置，不需要重複撰寫。

## 使用範例
以 Rolling 策略為例，我們可以模組化出兩種不同的策略：
- **asap-rolling-strategy**：盡快完成 Rolling，允許短暫的服務中斷
- safe-rolling-strategy：不允許服務中斷，新版本的 Pod 需 Ready 後才能關閉舊版 Pod。

### 準備 Components

> 代碼可參閱 [GitHub](https://github.com/YihongGao/iThome_30Day_2024/tree/main/resources/day18/kustomize-demo)

#### asap-rolling-strategy Component YAML
```yaml
# kustomize-demo/components/asap-rolling-strategy/api-rollingUpdate-strategy.yml
- op: replace
  path: /spec/strategy
  value: 
    rollingUpdate:
      maxSurge: 50%
      maxUnavailable: 100%

# kustomize-demo/components/asap-rolling-strategy/kustomization.yml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

patches:
- target:
    kind: Deployment
    name: .*-backend
  path: api-rollingUpdate-strategy.yml
```
> 📘 這裡使用了 JSON6902 Patch 與 模糊匹配，適用於所有名稱以 `-backend` 結尾的 `Deployment`，將其應用到 `api-rollingUpdate-strategy.yml` 中。

#### safe-rolling-strategy component YAML

```yaml
# kustomize-demo/components/safe-rolling-strategy/api-rollingUpdate-strategy.yml
- op: replace
  path: /spec/strategy
  value: 
    rollingUpdate:
      maxSurge: 50%
      maxUnavailable: 0

# kustomize-demo/components/safe-rolling-strategy/kustomization.yml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

patches:
- target:
    kind: Deployment
    name: .*-backend
  path: api-rollingUpdate-strategy.yml
```

這樣我們就定義了兩個 Components，路徑分別在
- `kustomize-demo/components/asap-rolling-strategy/kustomization.yml`（較鬆散的 Rolling 策略）
- `kustomize-demo/components/safe-rolling-strategy/kustomization.yml`（嚴謹的 Rolling 策略）

### 將 Components 套用到 Overlays
讓我們針對不同環境來套用 Components
- 開發環境(overlays/develop)：套用 `asap-rolling-strategy`
- 預生產環境(overlays/pre-production)：套用 `asap-rolling-strategy`
- 生產環境(overlays/production)：套用 `safe-rolling-strategy`

#### 開發環境(overlays/develop)
```yaml
# kustomization.yml
# 省略中間 apiVersion, kind, resource..等欄位

# 透過相對路徑載入 components
components:
  - ../../components/asap-rolling-strategy
```

#### 預生產環境(overlays/pre-production)
```yaml
# kustomization.yml
# 省略中間 apiVersion, kind, resource..等欄位

# 透過相對路徑載入 components
components:
  - ../../components/asap-rolling-strategy
```

#### 生產環境(overlays/production)
```yaml
# kustomization.yml
# 省略中間 apiVersion, kind, resource..等欄位

# 透過相對路徑載入 components
components:
  - ../../components/safe-rolling-strategy
```

在每個環境的 `kustomization.yml` 中透過 `components` 載入相應的策略，即可將指定的 Rolling 策略應用到該環境。接下來可以使用 `kustomize build` 來驗證每個環境的配置結果：
```shell
# develop
kustomize build overlays/develop | grep rollingUpdate -A 2
# output
    rollingUpdate:
      maxSurge: 50%
      maxUnavailable: 100%
--
    rollingUpdate:
      maxSurge: 50%
      maxUnavailable: 100%

# pre-production
kustomize build overlays/pre-production | grep rollingUpdate -A 2
# output
    rollingUpdate:
      maxSurge: 50%
      maxUnavailable: 100%
--
    rollingUpdate:
      maxSurge: 50%
      maxUnavailable: 100%

# production
kustomize build overlays/production | grep rollingUpdate -A 2
# output
    rollingUpdate:
      maxSurge: 50%
      maxUnavailable: 0
--
    rollingUpdate:
      maxSurge: 50%
      maxUnavailable: 0
```
這樣就成功將 Components 應用到了不同的環境中。

除了在多環境中進行配置差異化，Components 也可以用來減少 Base 中的重複配置，例如：
- **自動配置 Probes**：如果專案的服務有固定的 `Liveness/Readiness Probes` 路徑，可以通過 Components 將 Probes 配置應用到指定的 `Deployment` 上，無需在 `Base` 中重複定義，從而保持 `Base` YAML 的簡潔性和靈活性。

- **自動添加 Prometheus 服務發現所需的 Label**：若 Prometheus 使用特定的 Label 進行服務發現並收集 metrics，可以透過 Components 將這些 Label 配置到 Deployment 上。

### 自動配置 Probe 範例
```yaml
# components/spring-micrometer-monitor/kustomization.yml
apiVersion: kustomize.config.k8s.io/v1alpha1  
kind: Component

patches:
- target:
    kind: Deployment
    labelSelector: monitor=spring-micrometer-monitor
  path: probes.yml

# probes.yml
- op: replace
  path: /spec/template/spec/containers/0/startupProbe
  value: 
    failureThreshold: 30
    periodSeconds: 10
    httpGet:
      path: /q/health/liveness
      port: 9000
- op: replace
  path: /spec/template/spec/containers/0/livenessProbe
  value: 
    initialDelaySeconds: 30
    failureThreshold: 3
    periodSeconds: 20
    httpGet:
      path: /q/health/liveness
      port: 9000
- op: replace
  path: /spec/template/spec/containers/0/readinessProbe
  value: 
    periodSeconds: 5
    httpGet:
      path: /q/health/readiness
      port: 9000
```

### 自動添加 Prometheus 服務發現所需的 Label 範例
```yaml
# components/spring-micrometer-monitor/kustomization.yml
apiVersion: kustomize.config.k8s.io/v1alpha1  
kind: Component

patches:
- target:
    kind: Deployment
    labelSelector: monitor=spring-micrometer-monitor
  path: micrometer-monitor.yml

# scrape-metrics.yml
# 假設 prometheus 會依照以下 labels 進行服務發現並收集 metrics
# 透過 components 配置到 Deployment 上
- op: add
  path: /spec/template/spec/containers/0/ports/-
  value:
    name: metrics
    containerPort: 9000
    protocol: TCP
- op: add
  path: /spec/template/metadata/labels/gmp-prometheus.io~1scrape
  value: "true"
- op: add
  path: /spec/template/metadata/labels/monitor
  value:  micrometer-monitor
```

透過 Kustomize 的 Components，我們可以有效地模組化配置並提升重用性。這使得基礎資源（Base YAML）保持簡潔，並能透過 Components 進行擴展和功能的組合。最後，根據環境需求使用 patches 進行自訂配置，實現一個靈活且高可維護性的 YAML 管理策略。

# 小結
透過 Kustomize 改善 YAML 管理後，明天我們將介紹如何使用 Argo CD 實現 Pull-based 的 GitOps，進一步加強 Kubernetes 與 Manifest 檔案之間的一致性。

# Refernce
- [Kustomize 官方文件](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patches/#name-and-kind-changes)
- [christopher-adamson / Patches in Kustomize](https://www.linkedin.com/pulse/patches-kustomize-christopher-adamson-gaq4c)
- [openanalytics / kustomize-best-practices](https://www.openanalytics.eu/blog/2021/02/23/kustomize-best-practices/)