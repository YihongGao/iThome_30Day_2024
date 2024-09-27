
# Day-26 服務的部署策略 - Argo Rollouts with Ingress Nginx Controller

# 前言
昨天介紹 Argo rollouts 中的藍綠部署功能，今天將使用 Ingress NGINX Controller 作為 Argo rollouts **Traffic Management** 的實現，來實現**金絲雀部署（Canary）**
> 📘 若未搭配 **Traffic Management** 時，無法精確控制流量，僅能使用有限的 Canary 功能，細節可參閱 [官方文件](https://argoproj.github.io/argo-rollouts/features/canary/#overview)

## 環境準備
- **安裝 Ingress NGINX Controller**

  安裝方式參考： Day24 / 服務的部署策略 - Ingress NGINX Controller

- **安裝 Argo Rollouts**

  安裝方式參考：Day25 / 服務的部署策略 - Argo Rollouts

# 部署 Demo 服務
執行以下指令，將此 [Github repo](https://github.com/YihongGao/iThome_30Day_2024/blob/main/resources/day26/apps/canary/deploy.yml) 的 Demo 服務部署到 `ithome` namespeace
```shell
kubectl create namespace ithome
kubectl apply -f https://raw.githubusercontent.com/YihongGao/iThome_30Day_2024/refs/heads/main/resources/day26/apps/canary/deploy.yml
```

部署後會有以下資源：
- 1 個 Rollout： `rollouts.argoproj.io/app-backend` 代表我們正在運行的服務。
- 2 個 Service： `service/app-backend-stable`、`app-backend-preview` 穩定版本/新版本的 Service
- 2 個 Ingress： 
  - `primary-ingress`：負責穩定版本的 Ingress
  - `app-backend-primary-ingress-canary`：由 Argo Rollout 自動建立的 Ingress

先來看 Rollout 有什麼改變
> [Rollout 的 YAML](https://github.com/YihongGao/iThome_30Day_2024/blob/main/resources/day26/apps/canary/rollout.yml)的內容
```yaml
  strategy:
    canary:
      canaryService: app-backend-preview  # required
      stableService: app-backend-stable  # required
      trafficRouting:
        nginx:
          stableIngress: primary-ingress
          additionalIngressAnnotations:   # optional
            canary-by-header: Canary
            canary-by-header-value: "true"
      steps: # Deploy 
      - setWeight: 33
      - pause: {}
      - setWeight: 66
      - pause: {}
```
主要欄位介紹
  - `stableService`、`canaryService`：用於穩定版本/新版本的 Pod，作為流量入口的 Kubernetes Service。
  - `trafficRouting`：定義 **Traffic Management** 配置
    - `nginx`：使用 Ingress NGINX controller 作為 **Traffic Management** 的實現
      - `stableIngress`：
      - `additionalIngressAnnotations` 定義要添加到 Ingress 的 Annotation 
  - `steps`：定義多個發佈階段
    - `setWeight`：該發布階段將多少百分比的請求導向新版本
    - `pause`：暫停階段，到該階段時，需人工進行 `Promote` 才能進到下一個階段。

這個配置與昨日藍綠部署一樣有以下行為：
- 流量全部切換到新版本之前，都會同時運行新舊版本的 Pod
- 會透過 stableService 和 canaryService 將流量導向相應的版本。

而透過 `trafficRouting.nginx` 能交由 Argo Rollout 協助管理 Nginx Ingress 並控制流量轉導規則，不需再自己管理多組 Ingress 配置。

使用 `kubectl get ingress` 能看到 Argo Rollout 自動產生了一個 Ingress，該 Ingress 的名稱格式為：`<ROLLOUT-NAME>-<INGRESS-NAME>-canary`
```shell
kubectl get ingress
NAME                                 CLASS    HOSTS              ADDRESS     PORTS   AGE
app-backend-primary-ingress-canary   <none>   day24.ithome.com   localhost   80      54m
primary-ingress                      <none>   day24.ithome.com   localhost   80      54m
```

另外使用 `steps` 能定義你的發布計畫，後續交由自動/手動的方式逐漸發布新版本。



# 小結

# Refernce
- [Ingress NGINX Controller](https://github.com/kubernetes/ingress-nginx/tree/main)
- [Argo Rollout](https://argo-rollouts.readthedocs.io/en/stable/)