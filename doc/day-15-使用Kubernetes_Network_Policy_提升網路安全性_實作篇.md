
# Day-15 使用 Kubernetes NetworkPolicy 提升網路安全性 - 實作篇

# 前言
昨天介紹了 NetworkPolicy 的使用方式，但 kind 預設的 CNI 不支援 NetworkPolicy，所以今天我們將來安裝 [Cilium] 作為 CNI，並透過 [Hubble] 的 Service Map 來觀察 NetworkPolicy 的行為和網路流量。

# 環境準備
- 建立/重建 kind 環境，並禁用預設 CNI
- 安裝 [Cilium] 與 [Hubble]
- 部署 Demo 服務
  - Frontend Pod
  - Backend Pod
  - DB Pod

## 建立/重建 kind 環境，並禁用預設 CNI
> 代碼能參閱 [GitHub](https://github.com/YihongGao/iThome_30Day_2024/tree/main/resources/day15)

調整 kind-config.yml 禁用預設 CNI 
```yaml
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30000
    hostPort: 30000
    listenAddress: "0.0.0.0" # Optional, defaults to "0.0.0.0"
    protocol: tcp # Optional, defaults to tcp
  - containerPort: 30001
    hostPort: 30001
    listenAddress: "0.0.0.0" # Optional, defaults to "0.0.0.0"
    protocol: tcp # Optional, defaults to tcp
- role: worker
  labels:
    zone: local-a
- role: worker
  labels:
    zone: local-a
    GPU: "true"
- role: worker
  labels:
    zone: local-b
- role: worker
  labels:
    zone: local-b
networking:
  disableDefaultCNI: true
```

建立 cluster 
```shell
kind create cluster --config=kind-config.yaml  
```

## 安裝 [Cilium] 與 [Hubble]
依照[官方文件](https://docs.cilium.io/en/stable/installation/kind/#install-cilium)，透過 helm 安裝 [Cilium]

```shell
# Setup Helm repository:
helm repo add cilium https://helm.cilium.io/

# Preload the cilium image into each worker node in the kind cluster:
docker pull quay.io/cilium/cilium:v1.16.1
kind load docker-image quay.io/cilium/cilium:v1.16.1

# Then, install Cilium release via Helm:
helm install cilium cilium/cilium --version 1.16.1 \
   --namespace kube-system \
   --set image.pullPolicy=IfNotPresent \
   --set ipam.mode=kubernetes
```

安裝 cilium CLI ，用來驗證 CNI 配置是否成功
```shell
# Install cilium in MacOsS
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "arm64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-darwin-${CLI_ARCH}.tar.gz{,.sha256sum}
shasum -a 256 -c cilium-darwin-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-darwin-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-darwin-${CLI_ARCH}.tar.gz{,.sha256sum}
```
驗證 cilium 是否正常運作在 kind 建立的 k8s
```shell
cilium status --wait
```
看到以下畫面，能看到 `Cilium`、`Operator`、`Envoy DaemonSet` 狀態為 **OK** 即代表成功將 `Cilium` 安裝到 k8s 中。
```shell
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       disabled
    \__/       ClusterMesh:        disabled
# ... 以下省略
```

接著我們來啟用 Hubble 與 Hubble Web UI，後續要用來觀察網路流量時使用
```shell
helm upgrade cilium cilium/cilium --version 1.16.1 \
   --namespace kube-system \         
   --reuse-values \               
   --set hubble.relay.enabled=true \
   --set hubble.ui.enabled=true
```
再次透過 `cilium status` 指令，檢查 `Hubble Relay` 狀態應轉為 **OK**
```shell
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       OK
    \__/       ClusterMesh:        disabled
```

## 部署 Demo 服務
將部署三個 deployment，分別當作 three-tier 架構的每一層的其中一個服務
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240907/截圖-2024-09-07-下午10.36.16.70aaa5clyg.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240907/截圖-2024-09-07-下午10.36.16.70aaa5clyg.webp)

```shell
# 建立 namespace
kubectl create namespace ithome
# 切換 context namespace 到 ithome，之後執行 kubectl 會自帶 -n ithome
kubens ithome

# Install the deployment/Pod with tier=frontend
kubectl apply -f https://raw.githubusercontent.com/YihongGao/iThome_30Day_2024/main/resources/day15/workload/frontend.yml

# Install the deployment/Pod with tier=backend
kubectl apply -f https://raw.githubusercontent.com/YihongGao/iThome_30Day_2024/main/resources/day15/workload/backend.yml

# Install the deployment/Pod with tier=data
kubectl apply -f https://raw.githubusercontent.com/YihongGao/iThome_30Day_2024/main/resources/day15/workload/db.yml
```

# 驗證 NetworkPolicy
再開始配置 NetworkPolicy 之前，我們先開啟 hubble 的 Service Map 功能觀察 Pod 之間的網路流量。

## 開啟 hubble Service Map
```shell
cilium hubble ui
```
能在本地瀏覽器中開啟 Web UI，並選到 ithome 的 namespace
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240909/截圖-2024-09-09-下午4.57.02.7ax45tjose.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240909/截圖-2024-09-09-下午4.57.02.7ax45tjose.webp)

目前 UI 上還沒任何流量資訊，讓我們打一些流量到 frontend Pod 
```shell
kubectl exec -it ${you frontend pod name} -- curl localhost:80

# Response
{
  "message": "Hello from ithome demo data!"
}
```

再能看到 UI 上流量如何在 Pod 之間行進的，而下方也有抓取流量封包的詳細資訊
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240909/截圖-2024-09-09-下午5.55.39.4cku2de8c4.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240909/截圖-2024-09-09-下午5.55.39.4cku2de8c4.webp)

## 配置 NetworkPolicy

### 1. 配置預設 NetworkPolicy：拒絕全部進出流量
先配置拒絕全部進出流量的 NetworkPolicy 並向 frontend 打一個 Http request
```shell
kubectl apply https://raw.githubusercontent.com/YihongGao/iThome_30Day_2024/main/resources/day15/network-policy/default-deny-all.yml

kubectl exec -it ${you frontend pod name} -- curl localhost:80
```
等待一陣子後，Response 會出現 504 Gateway Time-out，代表 frontend Pod 連不上 backend Pod
``` html
<html>
<head><title>504 Gateway Time-out</title></head>
<body>
<center><h1>504 Gateway Time-out</h1></center>
<hr><center>nginx/1.27.1</center>
</body>
</html>
```
Hubble UI 也會展示哪一段網路連線失敗
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240909/截圖-2024-09-09-下午5.18.17.41y096ndw2.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240909/截圖-2024-09-09-下午5.18.17.41y096ndw2.webp)

以上符合我們的預期，因爲目前只配置了拒絕全部進出流量的 NetworkPolicy 到所有 Pod 上，接下來我們要逐步開放 Pod 之間的連線

### 開放 frontend 與 backend 之間流量通行
```shell
# frontend-tier-policy
kubectl apply -f https://raw.githubusercontent.com/YihongGao/iThome_30Day_2024/main/resources/day15/network-policy/frontend-tier-policy.yml

# backend-tier-policy
kubectl apply -f https://raw.githubusercontent.com/YihongGao/iThome_30Day_2024/main/resources/day15/network-policy/backend-tier-policy.yml

kubectl exec -it ${you frontend pod name} -- curl localhost:80
```

我們會發現一樣回應 504 Gateway Time-out，但能從 Hubble UI 發現是 backend 向 db 發出 http 連線時被阻擋了。
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240909/截圖-2024-09-09-下午5.46.05.6bh0sp9fsu.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240909/截圖-2024-09-09-下午5.46.05.6bh0sp9fsu.webp)

### 開放 backend 與 db 之間流量通行
```shell
kubectl apply -f https://raw.githubusercontent.com/YihongGao/iThome_30Day_2024/main/resources/day15/network-policy/data-tier-policy.yml

kubectl exec -it ${you frontend pod name} -- curl localhost:80

# Response
{
  "message": "Hello from ithome demo data!"
}
```
這次能看到請求成功了，並且能在 Hubble UI 看到 backend 到 db 之間的連線打通了。
![https://github.com/YihongGao/picx-images-hosting/raw/master/20240909/截圖-2024-09-09-下午4.56.07.5j45awyrcw.webp](https://github.com/YihongGao/picx-images-hosting/raw/master/20240909/截圖-2024-09-09-下午4.56.07.5j45awyrcw.webp)

# 小結
以上就是我們 NetworkPolicy 的實作，並透過基於[eBPF] 技術的觀測工具 [Hubble] 來檢視 Kubernetes 中的網路流量傳遞。

到目前為止，我們更了解 Kubernetes 的核心組件
明天會開始進入到 CI/CD pipeline 的實作章節，透過 Kustomize 和 ArgoCD 將 2023 年介紹的陽春版 CI/CD pipeline 升級為更易於維護的版本。

# Refernce
- [cilium 官方](https://cilium.io/)
- [eBPF 官方](https://ebpf.io/)

[Cilium]: https://docs.cilium.io/en/stable/overview/intro/

[Hubble]: https://docs.cilium.io/en/stable/overview/intro/#what-is-hubble

[eBPF]: https://ebpf.io/