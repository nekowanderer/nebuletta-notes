# Kubernetes Rolling Updates 實作

[English](../en/19_k8s_rolling_updates.md) | [繁體中文](../zh-tw/19_k8s_rolling_updates.md) | [日本語](../ja/19_k8s_rolling_updates.md) | [回到索引](../README.md)

## 前置準備

### 啟動 Docker 服務
```bash
$ sudo service docker start
$ docker ps
```

### 建立 Minikube Cluster
```bash
$ minikube start --driver docker
$ minikube status
```

## 部署初始版本 (v1)

### 建立 Deployment 定義檔
複製並建立 rolling update 的 deployment 檔案：

```bash
$ cp advanced-deployment.yaml advanced-deployment-rollingupdate.yaml
$ cat advanced-deployment-rollingupdate.yaml
```

### 部署 v1 版本
```bash
$ kubectl apply -f advanced-deployment-rollingupdate.yaml

$ kubectl get deployments
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
app-deployment   3/3     3            3           22s

$ kubectl get rs
NAME                        DESIRED   CURRENT   READY   AGE
app-deployment-7f57cfb4b6   3         3         3       24s
```

### 確認 Pod 狀態
```bash
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-7f57cfb4b6-5nmhg   1/1     Running   0          48s
app-deployment-7f57cfb4b6-qns74   1/1     Running   0          48s
app-deployment-7f57cfb4b6-stpv9   1/1     Running   0          48s

$ kubectl describe pod [pod_id] | grep Image:
    Image:          uopsdod/k8s-hostname-amd64-beta:v1
```

## 設定 Rolling Update 策略

### 修改 Deployment 配置
編輯 `advanced-deployment-rollingupdate.yaml` 檔案，加入 rolling update 策略並更新映像檔版本：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: app-pod
  template:
    metadata:
      labels:
        app: app-pod
    spec:
      containers:
      - name: app-container
        image: uopsdod/k8s-hostname-amd64-beta:v2
        imagePullPolicy: Always
        ports:
        - containerPort: 80
        startupProbe:
          httpGet:
            path: /healthcheck
            port: 80
          initialDelaySeconds: 0
          timeoutSeconds: 1
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 30
        livenessProbe:
          httpGet:
            path: /healthcheck
            port: 80
          initialDelaySeconds: 0
          timeoutSeconds: 1
          periodSeconds: 1
          successThreshold: 1
          failureThreshold: 5
        readinessProbe:
          httpGet:
            path: /healthcheck_dependency
            port: 80
          initialDelaySeconds: 0
          timeoutSeconds: 1
          periodSeconds: 1
          successThreshold: 5
          failureThreshold: 7
```

### 確認配置
```bash
$ cat advanced-deployment-rollingupdate.yaml
```

## 執行 Rolling Update

### 部署 v2 版本
```bash
$ kubectl apply -f advanced-deployment-rollingupdate.yaml
```

### 監控 Rolling Update 過程
在新的終端機視窗中執行以下命令來即時監控 ReplicaSet 變化：

```bash
$ kubectl get rs -w
NAME                        DESIRED   CURRENT   READY   AGE
app-deployment-7f57cfb4b6   3         3         3       3m12s
app-deployment-6f975d88d9   1         0         0       0s
app-deployment-6f975d88d9   1         0         0       0s
app-deployment-6f975d88d9   1         1         0       0s
app-deployment-6f975d88d9   1         1         1       14s
app-deployment-7f57cfb4b6   2         3         3       4m41s
app-deployment-6f975d88d9   2         1         1       14s
app-deployment-7f57cfb4b6   2         3         3       4m41s
app-deployment-6f975d88d9   2         1         1       14s
app-deployment-6f975d88d9   2         2         1       14s
app-deployment-7f57cfb4b6   2         2         2       4m41s
app-deployment-6f975d88d9   2         2         2       29s
app-deployment-7f57cfb4b6   1         2         2       4m56s
app-deployment-7f57cfb4b6   1         2         2       4m56s
app-deployment-6f975d88d9   3         2         2       29s
app-deployment-7f57cfb4b6   1         1         1       4m56s
app-deployment-6f975d88d9   3         2         2       29s
app-deployment-6f975d88d9   3         3         2       29s
app-deployment-6f975d88d9   3         3         3       44s
app-deployment-7f57cfb4b6   0         1         1       5m11s
app-deployment-7f57cfb4b6   0         1         1       5m11s
app-deployment-7f57cfb4b6   0         0         0       5m11s

$ kubectl get pods -w
NAME                              READY   STATUS              RESTARTS   AGE
app-deployment-6f975d88d9-t74gf   0/1     ContainerCreating   0          5s
app-deployment-7f57cfb4b6-5nmhg   1/1     Running             0          4m32s
app-deployment-7f57cfb4b6-qns74   1/1     Running             0          4m32s
app-deployment-7f57cfb4b6-stpv9   1/1     Running             0          4m32s
app-deployment-6f975d88d9-t74gf   0/1     Running             0          10s
app-deployment-6f975d88d9-t74gf   0/1     Running             0          11s
app-deployment-6f975d88d9-t74gf   1/1     Running             0          14s
app-deployment-7f57cfb4b6-stpv9   1/1     Terminating         0          4m41s
app-deployment-6f975d88d9-cq2xk   0/1     Pending             0          0s
app-deployment-6f975d88d9-cq2xk   0/1     Pending             0          0s
app-deployment-6f975d88d9-cq2xk   0/1     ContainerCreating   0          0s
app-deployment-6f975d88d9-cq2xk   0/1     Running             0          3s
app-deployment-6f975d88d9-cq2xk   0/1     Running             0          10s
app-deployment-6f975d88d9-cq2xk   1/1     Running             0          15s
app-deployment-7f57cfb4b6-qns74   1/1     Terminating         0          4m56s
app-deployment-6f975d88d9-2lwcb   0/1     Pending             0          0s
app-deployment-6f975d88d9-2lwcb   0/1     Pending             0          0s
app-deployment-6f975d88d9-2lwcb   0/1     ContainerCreating   0          0s
app-deployment-6f975d88d9-2lwcb   0/1     Running             0          3s
app-deployment-6f975d88d9-2lwcb   0/1     Running             0          10s
app-deployment-7f57cfb4b6-stpv9   0/1     Error               0          5m11s
app-deployment-6f975d88d9-2lwcb   1/1     Running             0          15s
app-deployment-7f57cfb4b6-5nmhg   1/1     Terminating         0          5m11s
app-deployment-7f57cfb4b6-stpv9   0/1     Error               0          5m11s
app-deployment-7f57cfb4b6-stpv9   0/1     Error               0          5m11s
app-deployment-7f57cfb4b6-qns74   0/1     Error               0          5m27s
app-deployment-7f57cfb4b6-5nmhg   0/1     Error               0          5m42s
```

### 確認更新結果
```bash
# 檢查 ReplicaSet 狀態
$ kubectl get rs
NAME                        DESIRED   CURRENT   READY   AGE
app-deployment-6f975d88d9   3         3         3       114s
app-deployment-7f57cfb4b6   0         0         0       6m21s

# 檢查 Pod 狀態
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-6f975d88d9-2lwcb   1/1     Running   0          81s
app-deployment-6f975d88d9-cq2xk   1/1     Running   0          96s
app-deployment-6f975d88d9-t74gf   1/1     Running   0          110s

# 確認 Pod 使用的映像檔版本
$ kubectl describe pod [pod_id] | grep Image:
    Image:          uopsdod/k8s-hostname-amd64-beta:v2
```

## Rolling Update 策略說明

### 策略參數
- **type: RollingUpdate**: 指定使用滾動更新策略
- **maxSurge: 1**: 允許超出期望副本數的最大數量（可同時運行的 Pod 數量）
- **maxUnavailable: 0**: 更新過程中不可用的 Pod 數量（0 表示不允許服務中斷）

### 更新流程
1. Kubernetes 會先建立新的 ReplicaSet 來部署 v2 版本
2. 逐步將舊版本的 Pod 替換為新版本
3. 確保服務在更新過程中持續可用

## 重要概念

- **Rolling Update**: 逐步更新 Pod 的策略，確保服務不中斷
- **maxSurge**: 控制更新過程中可以超出期望副本數的數量
- **maxUnavailable**: 控制更新過程中允許不可用的 Pod 數量
- **ReplicaSet 管理**: 每個版本會產生新的 ReplicaSet 來管理對應的 Pod
- **健康檢查**: 使用 probes 確保新 Pod 正常運作後才進行下一步更新
