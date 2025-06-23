# Kubernetes Readiness Probe 實作

[English](../en/18_readiness_probe.md) | [繁體中文](../zh-tw/18_readiness_probe.md) | [日本語](../ja/18_readiness_probe.md) | [回到索引](../README.md)

## 概述
Readiness Probe 用於檢查 Pod 是否準備好接收流量。當 Readiness Probe 失敗時，Pod 會從 Service 的負載平衡器中移除，但 Pod 本身不會被重啟。

## 前置準備

### 啟動 Docker 環境
```bash
$ sudo service docker start
$ docker ps
```

### 建立 Minikube Cluster
```bash
$ minikube start --driver docker
$ minikube status
```

## 建立 Deployment 配置

### 建立 YAML 檔案
```bash
$ vi advanced-deployment.yaml
```

### Deployment 配置內容
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
spec:
  replicas: 3
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
        image: uopsdod/k8s-hostname-amd64-beta:v1
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

### 檢視配置檔案
```bash
$ cat advanced-deployment.yaml
```

## 部署與監控

### 部署 Deployment
```bash
$ kubectl apply -f advanced-deployment.yaml
```

### 監控 ReplicaSet
```bash
$ kubectl get rs -w
NAME                        DESIRED   CURRENT   READY   AGE
app-deployment-7f57cfb4b6   3         3         0       10s
app-deployment-7f57cfb4b6   3         3         1       15s
app-deployment-7f57cfb4b6   3         3         2       15s
app-deployment-7f57cfb4b6   3         3         3       15s
```

### 監控 Pod 狀態
```bash
$ kubectl get pods -w
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-7f57cfb4b6-g5sz7   1/1     Running   0          26s
app-deployment-7f57cfb4b6-mjnzw   1/1     Running   0          26s
app-deployment-7f57cfb4b6-q6xdp   1/1     Running   0          26s
app-deployment-7f57cfb4b6-q6xdp   0/1     Running   0          3m33s
```

## 建立 Service

### 部署 NodePort Service
```bash
$ kubectl apply -f simple-service-nodeport.yaml
```

### 取得節點資訊
```bash
$ kubectl get node
NAME       STATUS   ROLES           AGE    VERSION
minikube   Ready    control-plane   2d8h   v1.33.1

$ kubectl describe node minikube | grep InternalIP
InternalIP:  192.168.49.2
```

### 取得 Service 資訊
```bash
$ kubectl get services
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
app-service-nodeport   NodePort    10.111.160.67   <none>        8080:30080/TCP   93s
kubernetes             ClusterIP   10.96.0.1       <none>        443/TCP          22h
```

## 測試 Readiness Probe

### 模擬健康檢查失敗
```bash
$ curl http://192.168.49.2:30080/healthcheck_dependency_switchstatus
[beta] served by: app-deployment-7f57cfb4b6-q6xdp.
[beta] isDependancyHealthy value switched to false.
```

### 監控 Pod 狀態變化
在新的終端視窗中執行：
```bash
$ kubectl get pods -w
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-7f57cfb4b6-g5sz7   1/1     Running   0          26s
app-deployment-7f57cfb4b6-mjnzw   1/1     Running   0          26s
app-deployment-7f57cfb4b6-q6xdp   1/1     Running   0          26s
app-deployment-7f57cfb4b6-q6xdp   0/1     Running   0          3m33s
```
預期結果：應該會看到 `Ready 0/1` 狀態

### 測試應用程式存取
```bash
# 不會看到來自 app-deployment-7f57cfb4b6-q6xdp 的 response
$ curl http://${NODE_IP}:${NODE_PORT}/
```

### 檢查 Pod 詳細狀態
```bash
$ kubectl describe pod [pod_id]
Events:
  Type     Reason     Age                 From               Message
  ----     ------     ----                ----               -------
  Normal   Scheduled  4m32s               default-scheduler  Successfully assigned default/app-deployment-7f57cfb4b6-q6xdp to minikube
  Normal   Pulling    4m32s               kubelet            Pulling image "uopsdod/k8s-hostname-amd64-beta:v1"
  Normal   Pulled     4m29s               kubelet            Successfully pulled image "uopsdod/k8s-hostname-amd64-beta:v1" in 1.456s (3.11s including waiting). Image size: 914594685 bytes.
  Normal   Created    4m29s               kubelet            Created container: app-container
  Normal   Started    4m28s               kubelet            Started container app-container
  Warning  Unhealthy  42s (x25 over 65s)  kubelet            Readiness probe failed: HTTP probe failed with statuscode: 500
```
預期結果：應該會看到 `Readiness probe failed: HTTP probe failed with statuscode: 500`

## 清理資源

### 刪除所有部署的資源
```bash
$ kubectl delete deployments --all
$ kubectl delete services --all
```

## 重要說明

- **Readiness Probe**：檢查 Pod 是否準備好接收流量
- **Liveness Probe**：檢查 Pod 是否存活，失敗時會重啟 Pod
- **Startup Probe**：在 Pod 啟動期間使用，完成後會切換到 Liveness Probe
- 當 Readiness Probe 失敗時，Pod 會從 Service 的負載平衡器中移除，但不會重啟

