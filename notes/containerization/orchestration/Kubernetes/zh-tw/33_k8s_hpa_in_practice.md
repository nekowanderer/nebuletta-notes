# Kubernetes HPA (Horizontal Pod Autoscaler) 實作

[English](../en/33_k8s_hpa_in_practice.md) | [繁體中文](../zh-tw/33_k8s_hpa_in_practice.md) | [日本語](../ja/33_k8s_hpa_in_practice.md) | [回到索引](../README.md)

## 概述
本筆記將展示如何在 Kubernetes 中設定和使用 Horizontal Pod Autoscaler (HPA)，實現基於 CPU 使用率的自動擴縮容功能。

## 前置準備

### 1. 啟動 Docker 環境
```bash
$ sudo service docker start
$ docker ps
```

### 2. 建立 Minikube 叢集
```bash
$ minikube start --driver docker
$ minikube status
```

### 3. 啟用 Ingress 附加元件
```bash
$ minikube addons enable ingress

$ kubectl get all -n ingress-nginx
NAME                                           READY   STATUS      RESTARTS      AGE
pod/ingress-nginx-admission-create-5fj8c       0/1     Completed   0             46h
pod/ingress-nginx-admission-patch-529hj        0/1     Completed   1             46h
pod/ingress-nginx-controller-67c5cb88f-dv4px   1/1     Running     2 (48m ago)   46h

NAME                                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx-controller             NodePort    10.102.245.181   <none>        80:30447/TCP,443:31545/TCP   46h
service/ingress-nginx-controller-admission   ClusterIP   10.96.249.142    <none>        443/TCP                      46h

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-nginx-controller   1/1     1            1           46h

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-nginx-controller-67c5cb88f   1         1         1       46h

NAME                                       STATUS     COMPLETIONS   DURATION   AGE
job.batch/ingress-nginx-admission-create   Complete   1/1           7s         46h
job.batch/ingress-nginx-admission-patch    Complete   1/1           7s         46h
```

## 部署應用程式

### 1. 建立 HPA 版本的應用程式配置
複製並修改現有的應用程式配置檔案：

```bash
$ cp beta-app-all.yaml beta-app-all-hpa.yaml
$ vi beta-app-all-hpa.yaml
```

### 2. 應用程式配置檔案內容
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app-ns
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: beta-app-deployment
  namespace: app-ns
spec:
  replicas: 1  # 改成 1
  selector:
    matchLabels:
      app: beta-app-pod
  template:
    metadata:
      labels:
        app: beta-app-pod
    spec:
      containers:
      - name: beta-app-container
        image: uopsdod/k8s-hostname-amd64-beta:v1
        ports:
        - containerPort: 80
        resources:     # 加入這個 section
          limits:
            cpu: 300m
          requests:
            cpu: 200m
---
apiVersion: v1
kind: Service
metadata:
  name: beta-app-service-clusterip
  namespace: app-ns
spec:
  type: ClusterIP      # 加入這個
  selector:
    app: beta-app-pod
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
```

**重要變更：**
- 設定 `replicas: 1` 作為初始副本數
- 新增 `resources` 區段，包含 CPU 限制和請求
- CPU 限制設為 `300m`，請求設為 `200m`

### 3. 部署應用程式
```bash
$ kubectl apply -f beta-app-all-hpa.yaml
namespace/app-ns created
deployment.apps/beta-app-deployment created
service/beta-app-service-clusterip created

$ kubectl get all -n app-ns
NAME                                       READY   STATUS    RESTARTS   AGE
pod/beta-app-deployment-8679d5777b-5kzjj   1/1     Running   0          9s

NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/beta-app-service-clusterip   ClusterIP   10.105.44.132   <none>        8080/TCP   9s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/beta-app-deployment   1/1     1            1           9s

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/beta-app-deployment-8679d5777b   1         1         1       9s
```

### 4. 部署生產環境應用程式
```bash
$ kubectl apply -f prod-app-all.yaml
namespace/app-ns unchanged
deployment.apps/prod-app-deployment created
service/prod-app-service-clusterip created

$ kubectl get all -n app-ns
NAME                                       READY   STATUS    RESTARTS   AGE
pod/beta-app-deployment-8679d5777b-5kzjj   1/1     Running   0          36s
pod/prod-app-deployment-586c5dcc59-b88dw   1/1     Running   0          10s
pod/prod-app-deployment-586c5dcc59-fc82v   1/1     Running   0          10s
pod/prod-app-deployment-586c5dcc59-fdtdq   1/1     Running   0          10s

NAME                                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/beta-app-service-clusterip   ClusterIP   10.105.44.132    <none>        8080/TCP   36s
service/prod-app-service-clusterip   ClusterIP   10.100.206.114   <none>        8080/TCP   10s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/beta-app-deployment   1/1     1            1           36s
deployment.apps/prod-app-deployment   3/3     3            3           10s

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/beta-app-deployment-8679d5777b   1         1         1       36s
replicaset.apps/prod-app-deployment-586c5dcc59   3         3         3       10s
```

### 5. 部署 Ingress 路由
```bash
$ kubectl apply -f ingress-path.yaml

$ kubectl get ingress -n app-ns

$ kubectl get ingress -n app-ns -w
NAME           CLASS   HOSTS          ADDRESS   PORTS   AGE
ingress-path   nginx   all.demo.com             80      19s
ingress-path   nginx   all.demo.com   192.168.49.2   80      33s
```

## 設定 Metrics Server

### 1. 下載 Metrics Server 配置
```bash
$ wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
$ mv components.yaml metrics-server.yaml
```

### 2. 修改 Metrics Server 配置
```bash
$ vi metrics-server.yaml
```

在容器參數中新增 `--kubelet-insecure-tls` 參數：

```yaml
spec:
  containers:
  - args:
    - --cert-dir=/tmp
    - --secure-port=10250
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --kubelet-use-node-status-port
    - --metric-resolution=15s
    - --kubelet-insecure-tls  # 加入這行就好
    image: k8s.gcr.io/metrics-server/metrics-server:v0.7.2
```

### 3. 部署 Metrics Server
```bash
$ kubectl apply -f metrics-server.yaml
$ kubectl get deployment metrics-server -n kube-system
$ kubectl get deployment metrics-server -n kube-system -w
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
metrics-server   0/1     1            0           19m
metrics-server   1/1     1            1           19m

# 如果一直起不來的話，可以用這行看 log
$ kubectl logs -n kube-system deployment/metrics-server
```

## 設定 HPA

### 1. 建立 HPA
```bash
$ kubectl autoscale deployment beta-app-deployment --cpu-percent=10 --min=1 --max=10 -n app-ns
horizontalpodautoscaler.autoscaling/beta-app-deployment autoscaled

$ kubectl get hpa -n app-ns

$ kubectl get hpa -n app-ns -w
NAME                  REFERENCE                        TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%   1         10        1          78s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%   1         10        1          2m45s
```

**HPA 參數說明：**
- `--cpu-percent=10`：當 CPU 使用率超過 10% 的 pod cpu requests 時觸發擴容，以這邊的範例來說：
  - HPA threshold = CPU requests x HPA percentage = 200 x 0.1 = 20m
- `--min=1`：最小副本數為 1
- `--max=10`：最大副本數為 10
- 更多參數相關說明請參照下方的 `CPU 資源配置說明` 區塊

## 測試自動擴縮容

### 1. 測試自動擴容 (Scale Up)
開啟新的終端機視窗：

```bash
$ kubectl get ingress -n app-ns
$ ingress_ip=192.168.49.2
$ curl ${ingress_ip}:80/beta -H 'Host: all.demo.com'
$ while sleep 0.005; do curl -s ${ingress_ip}:80/beta -H 'Host: all.demo.com'; done
```

這個指令會持續發送請求到應用程式，當 CPU 使用率超過 10% 時，HPA 會自動增加 Pod 副本數。

### 2. 測試自動縮容 (Scale Down)
停止負載測試：
```bash
# 按 Ctrl + C 停止持續請求
```

停止負載後，HPA 會監控 CPU 使用率，當使用率降低時會自動減少 Pod 副本數。

## 監控和驗證

### 查看 HPA 狀態
```bash
$ kubectl describe hpa beta-app-deployment -n app-ns
Name:                                                  beta-app-deployment
Namespace:                                             app-ns
Labels:                                                <none>
Annotations:                                           <none>
CreationTimestamp:                                     Tue, 01 Jul 2025 14:07:28 +0000
Reference:                                             Deployment/beta-app-deployment
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  9% (19m) / 10%
Min replicas:                                          1
Max replicas:                                          10
Deployment pods:                                       3 current / 3 desired
Conditions:
  Type            Status  Reason              Message
  ----            ------  ------              -------
  AbleToScale     True    ReadyForNewScale    recommended size matches current size
  ScalingActive   True    ValidMetricFound    the HPA was able to successfully calculate a replica count from cpu resource utilization (percentage of request)
  ScalingLimited  False   DesiredWithinRange  the desired count is within the acceptable range
Events:
  Type    Reason             Age   From                       Message
  ----    ------             ----  ----                       -------
  Normal  SuccessfulRescale  62s   horizontal-pod-autoscaler  New size: 3; reason: cpu resource utilization (percentage of request) above target

# 關掉 curl loop 約五分鐘後：
$ kubectl get hpa -n app-ns
NAME                  REFERENCE                        TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%   1         10        1          78s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%   1         10        1          2m45s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%   1         10        1          3m
beta-app-deployment   Deployment/beta-app-deployment   cpu: 24%/10%   1         10        1          4m30s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 14%/10%   1         10        3          4m45s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 10%/10%   1         10        3          5m
beta-app-deployment   Deployment/beta-app-deployment   cpu: 9%/10%    1         10        3          5m15s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 8%/10%    1         10        3          5m45s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%    1         10        3          6m
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%    1         10        3          6m16s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%    1         10        3          6m31s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%    1         10        3          8m46s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%    1         10        3          10m
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%    1         10        1          11m


$ kubectl describe hpa beta-app-deployment -n app-ns
Name:                                                  beta-app-deployment
Namespace:                                             app-ns
Labels:                                                <none>
Annotations:                                           <none>
CreationTimestamp:                                     Tue, 01 Jul 2025 14:07:28 +0000
Reference:                                             Deployment/beta-app-deployment
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  0% (0) / 10%
Min replicas:                                          1
Max replicas:                                          10
Deployment pods:                                       3 current / 1 desired
Conditions:
  Type            Status  Reason            Message
  ----            ------  ------            -------
  AbleToScale     True    SucceededRescale  the HPA controller was able to update the target scale to 1
  ScalingActive   True    ValidMetricFound  the HPA was able to successfully calculate a replica count from cpu resource utilization (percentage of request)
  ScalingLimited  True    TooFewReplicas    the desired replica count is less than the minimum replica count
Events:
  Type    Reason             Age    From                       Message
  ----    ------             ----   ----                       -------
  Normal  SuccessfulRescale  6m17s  horizontal-pod-autoscaler  New size: 3; reason: cpu resource utilization (percentage of request) above target
  Normal  SuccessfulRescale  1s     horizontal-pod-autoscaler  New size: 1; reason: All metrics below target
```

## 資源清理

### 清理所有資源
```bash
$ kubectl delete namespace app-ns
```

## HPA 建立方式說明

### 本實驗的 HPA 建立方式
本實驗使用 `kubectl autoscale` 指令建立 HPA，這會使用 Kubernetes 的[預設行為](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#default-behavior)設定：
- **縮容穩定時間**：5 分鐘 (`stabilizationWindowSeconds: 300`)
- **擴容穩定時間**：馬上 (`stabilizationWindowSeconds: 0`)

### 自訂 HPA 配置
如果需要調整 HPA 行為，可以建立自訂的 HPA YAML 檔案：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: beta-app-deployment
  namespace: app-ns
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: beta-app-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 10
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 600  # 自訂縮容穩定時間
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 180  # 自訂擴容穩定時間
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
```

### 修改現有 HPA
也可以使用 patch 指令修改現有的 HPA：

```bash
$ kubectl patch hpa beta-app-deployment -n app-ns --type='merge' -p='
spec:
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 60
'
```

## CPU 資源配置說明

#### CPU 單位說明
- **1 CPU = 1000m (millicores)**
- **300m = 0.3 CPU = 30% 的單一 CPU 核心**
- **200m = 0.2 CPU = 20% 的單一 CPU 核心**

#### CPU Requests (請求) - 200m
```yaml
resources:
  requests:
    cpu: 200m
```

**意義：**
- **資源保證**：Kubernetes 保證為 Pod 分配至少 200m 的 CPU 資源
- **排程依據**：Scheduler 只會將 Pod 排程到有足夠 CPU 資源的節點上
- **HPA 計算基準**：HPA 使用 requests 值作為 CPU 使用率百分比的計算基準
  - 為什麼這樣設計？
    - 相對性: 使用百分比而不是絕對值，讓 HPA 適應不同大小的 Pod
    - 靈活性: 同一個 HPA 設定可以適用於不同資源配置的應用程式
    - 可預測性: 基於 requests 計算，確保行為一致
- **資源預留**：即使 Pod 沒有使用 CPU，這些資源也會被預留，不會分配給其他 Pod

#### CPU Limits (限制) - 300m
```yaml
resources:
  limits:
    cpu: 300m
```

**意義：**
- **資源上限**：Pod 最多只能使用 300m 的 CPU 資源
- **節流機制**：當 Pod 嘗試使用超過 300m 的 CPU 時，會被節流 (throttling)
- **防止資源濫用**：避免單一 Pod 佔用過多 CPU 資源，影響其他 Pod
- **可突發使用**：Pod 可以在短時間內使用超過 requests 但低於 limits 的資源

#### 實際運作範例
- requests: 200m
- limits: 300m

| 負載 | CPU 使用量 | 計算結果 | HPA 行為 | 系統狀態 |
|------|------------|----------|----------|----------|
| 🔵 **低** | 50m CPU | 50m ÷ 200m = 25% | 不會觸發 HPA 縮容 | 節點上仍有 150m CPU 被預留但未使用 |
| 🟢 **正常** | 150m CPU | 150m ÷ 200m = 75% | 如果閾值設為 70%，會觸發擴容 | Pod 正常運作，資源充足 |
| 🔴 **高** | 嘗試使用 400m CPU | 實際只能使用 300m (limits 限制) | 超出部分被節流，可能導致效能下降 | Pod 效能受限，需要調整 limits |

#### 設定建議

**Requests 設定原則：**
- 設定為應用程式的平均 CPU 使用率
- 過低：可能導致 Pod 無法獲得足夠資源
- 過高：會浪費節點資源，降低叢集利用率

**Limits 設定原則：**
- 通常設為 requests 的 1.5-2 倍
- 過低：可能限制應用程式的效能
- 過高：失去資源保護的作用

**HPA 考量：**
- HPA 基於 requests 計算百分比
- 設定 `--cpu-percent=10` 表示當 CPU 使用率超過 requests 的 10% 時觸發擴容
- 在這個例子中，就是當 CPU 使用率超過 20m (200m × 10%) 時觸發擴容

## 重要注意事項

1. **Metrics Server 必須正常運作**：HPA 依賴 Metrics Server 來獲取 CPU 和記憶體使用率指標
2. **資源限制設定**：必須在 Deployment 中設定 `resources.requests` 和 `resources.limits`
3. **CPU 百分比計算**：HPA 基於 `requests` 值計算 CPU 使用率百分比
4. **冷卻時間**：HPA 有內建的冷卻時間（約五分鐘），避免頻繁的擴縮容操作
5. **監控間隔**：預設每 15 秒檢查一次指標

## 故障排除

### 常見問題
1. **HPA 無法獲取指標**：檢查 Metrics Server 是否正常運作
2. **擴縮容不生效**：確認 Deployment 中已設定資源限制
3. **Ingress 無法存取**：檢查 Ingress 控制器狀態和網路配置

