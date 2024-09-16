
# Day-18 Kustomize 介紹 - Components

# 前言
昨天介紹了如何在單一個 Overlay 的 `Kustomization.yaml` 中透過 Patch 調整 YAML。

而今天要介紹的 Components 是 Kustomize 中一個更高級的概念，能將 Resource 的片段進行 **模組化**，並讓多個 Overlays 共享，提供可重用性與維護性。

## 為什麼要使用 Components？
實務上的環境管理可能有更複雜的情境出現，例如
- 開發環境：較少的安全限制，較鬆散的 Rolling 策略
- 預生產環境：完整的安全限制，較鬆散的 Rolling 策略
- 生產環境：完整的安全限制，嚴謹的 Rolling 策略
這時將 安全限制 與 Rolling 策略的配置，透過 Componentes 進行模組化，就於指定的環境使用該配置，不需要重複撰寫。

## 使用範例
用 Rolling 策略 作為使用範例，我們模組化出兩種 Rolling 策略
- asap-rolling-strategy：盡快完成 Rolling，允許服務中斷
- safe-rolling-strategy：不允許服務中斷，新版本的 Pod 需 Ready 才關閉舊 Pod

### 準備 Components
#### asap-rolling-strategy component YAML
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
> 📘 這邊使用了 JSON6902 Patch 搭配模糊匹配的方式，只要是 Deployment 且 name 為 -backend結尾都會套用到 `api-rollingUpdate-strategy.yml`

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

### 將 Components 套用到 Overlays
- 開發環境(overlays/develop)：套用 asap-rolling-strategy
- 預生產環境(overlays/pre-production)：套用 asap-rolling-strategy
- 生產環境(overlays/production)：套用 safe-rolling-strategy

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

只要在 overlays 的 kustomization.yml 透過 `components` 載入 components 就能將指定的 Rolling 策略套用近來，我們能使用 `kustomize build` 驗證結果，能看到每個環境套用到指定的 rollingUpdate 配置了
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
以上就是 Components 的使用方式。

不只在環境有差異的時候能使用，我們也能利用 Components 減少 Base 中重複的配置，比如：    
- 自動配置 Probes：假設專案的服務有固定的 `Liveness Probes`, `Readiness Probes` 路徑，能透過 Components 將 Probes 配置套用到指定的 Deployment 上，不用重複配置在 base YAML 中，能保持 base 的 YAML 簡單且具有彈性。
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

- 自動添加 Prometheus 服務發現需要的 Label
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

# 小結
透過 components，Kustomize 能有效處理配置的模組化與可重用性問題。它允許我們將基礎資源（base YAML）保持簡單，並透過 components 來進行組合與功能擴展。最後，使用 patches 為特定環境進行自訂，實現一個靈活且具備高維護性的 YAML 管理策略。


# Refernce
- [Kustomize 官方文件](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patches/#name-and-kind-changes)
- [christopher-adamson / Patches in Kustomize](https://www.linkedin.com/pulse/patches-kustomize-christopher-adamson-gaq4c)
- [openanalytics / kustomize-best-practices](https://www.openanalytics.eu/blog/2021/02/23/kustomize-best-practices/)