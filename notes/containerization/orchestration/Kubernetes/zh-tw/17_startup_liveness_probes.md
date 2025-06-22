# Kubernetes Startup 與 Liveness Probe 實作

[English](../en/17_startup_liveness_probes.md) | [繁體中文](../zh-tw/17_startup_liveness_probes.md) | [日本語](../ja/17_startup_liveness_probes.md) | [回到索引](../README.md)

本文件將示範如何在 Kubernetes 中實作 Startup Probe 和 Liveness Probe，並透過實際操作來觀察探針的行為。

### 前置準備

##### 1. 啟動 Docker 環境
```bash
$ sudo service docker start
$ docker ps
```

##### 2. 建立 Minikube Cluster
```bash
$ minikube start --driver docker
$ minikube status
```

### 建立 Deployment 配置

重複利用 [10_deploy_deployments](./10_deploy_deployments.md) 裡面的 simple-deployment.yml

##### 1. 複製基礎配置
```bash
$ cp simple-deployment.yaml advanced-deployment.yaml
```

##### 2. 編輯配置檔案
使用 `vi advanced-deployment.yaml` 編輯檔案，加入 Startup Probe 和 Liveness Probe 配置：

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
        
        # Startup Probe: 給予應用程式啟動時間
        startupProbe:
          httpGet:
            path: /healthcheck
            port: 80
          initialDelaySeconds: 0
          timeoutSeconds: 1
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 30
        
        # Liveness Probe: 監控應用程式是否存活
        livenessProbe:
          httpGet:
            path: /healthcheck
            port: 80
          initialDelaySeconds: 0
          timeoutSeconds: 1
          periodSeconds: 1
          successThreshold: 1
          failureThreshold: 5
```

##### 3. 確認配置
```bash
$ cat advanced-deployment.yaml
```

### 部署與監控

##### 1. 部署 Deployment
```bash
$ kubectl apply -f advanced-deployment.yaml
```

##### 2. 監控 ReplicaSet
```bash
$ kubectl get rs -w
NAME                        DESIRED   CURRENT   READY   AGE
app-deployment-6786b5648d   3         3         0       8s
app-deployment-6786b5648d   3         3         1       11s
app-deployment-6786b5648d   3         3         2       11s
app-deployment-6786b5648d   3         3         3       11s
```

##### 3. 部署 Service
```bash
$ kubectl apply -f simple-service-nodeport.yaml
```

### 測試探針功能

##### 1. 取得節點資訊
```bash
$ kubectl get node
$ kubectl describe node minikube
...
Addresses:
  InternalIP:  192.168.49.2
  Hostname:    minikube
...

# 設定節點 IP（請根據實際輸出調整）
$ NODE_IP=192.168.49.2
```

##### 2. 取得服務資訊
```bash
$ kubectl get services
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
app-service-nodeport   NodePort    10.104.200.155   <none>        8080:30080/TCP   54s
kubernetes             ClusterIP   10.96.0.1        <none>        443/TCP          7h54m

# 設定 NodePort（請根據實際輸出調整）
$ NODE_PORT=3XXXX
```

##### 3. 模擬健康檢查失敗
```bash
$ curl http://${NODE_IP}:${NODE_PORT}/healthcheck_switchstatus
[beta] served by: app-deployment-6786b5648d-757bq.
[beta] isHealthy value switched to false.
```

##### 4. 監控 Pod 狀態
```bash
$ kubectl get pods -w
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-6786b5648d-757bq   1/1     Running   0          3m42s
app-deployment-6786b5648d-jlvdw   1/1     Running   0          3m42s
app-deployment-6786b5648d-psqph   1/1     Running   0          3m42s
app-deployment-6786b5648d-757bq   0/1     Running   1 (2s ago)   4m9s
app-deployment-6786b5648d-757bq   0/1     Running   1 (3s ago)   4m10s
app-deployment-6786b5648d-757bq   1/1     Running   1 (3s ago)   4m10s

# 查看特定 Pod 的詳細資訊
$ kubectl describe pod [pod_id]
...
Events:
  Type     Reason     Age                  From               Message
  ----     ------     ----                 ----               -------
  Normal   Scheduled  4m36s                default-scheduler  Successfully assigned default/app-deployment-6786b5648d-757bq to minikube
  Normal   Pulled     4m35s                kubelet            Successfully pulled image "uopsdod/k8s-hostname-amd64-beta:v1" in 1.663s (1.663s including waiting). Image size: 914594685 bytes.
  Warning  Unhealthy  60s (x5 over 64s)    kubelet            Liveness probe failed: HTTP probe failed with statuscode: 500
  Normal   Killing    60s                  kubelet            Container app-container failed liveness probe, will be restarted
  Normal   Pulling    30s (x2 over 4m36s)  kubelet            Pulling image "uopsdod/k8s-hostname-amd64-beta:v1"
  Normal   Created    28s (x2 over 4m35s)  kubelet            Created container: app-container
  Normal   Started    28s (x2 over 4m35s)  kubelet            Started container app-container
  Normal   Pulled     28s                  kubelet            Successfully pulled image "uopsdod/k8s-hostname-amd64-beta:v1" in 1.462s (1.462s including waiting). Image size: 914594685 bytes.
...
```

### 清理資源

```bash
$ kubectl delete deployments --all
$ kubectl delete services --all
```

### 探針配置說明

##### Startup Probe 參數
- **`initialDelaySeconds: 0`**: 容器啟動後立即開始探測
- **`timeoutSeconds: 1`**: 每次探測的超時時間為 1 秒
- **`periodSeconds: 10`**: 每 10 秒執行一次探測
- **`successThreshold: 1`**: 成功 1 次即認為啟動完成
- **`failureThreshold: 30`**: 最多允許 30 次失敗（總共 5 分鐘啟動時間）

##### Liveness Probe 參數
- **`initialDelaySeconds: 0`**: 容器啟動後立即開始探測
- **`timeoutSeconds: 1`**: 每次探測的超時時間為 1 秒
- **`periodSeconds: 1`**: 每 1 秒執行一次探測
- **`successThreshold: 1`**: 成功 1 次即認為存活
- **`failureThreshold: 5`**: 最多允許 5 次失敗（5 秒後重啟容器）

### 預期結果

1. **啟動階段**: Startup Probe 會給予應用程式足夠的啟動時間
2. **運行階段**: Liveness Probe 會持續監控應用程式狀態
3. **故障處理**: 當健康檢查失敗時，Kubernetes 會自動重啟容器
4. **監控驗證**: 可以透過 `kubectl describe pod` 查看探針的詳細執行記錄
