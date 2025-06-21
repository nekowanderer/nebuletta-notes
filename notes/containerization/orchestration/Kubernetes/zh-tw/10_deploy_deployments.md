# Kubernetes Deployment 部署實作

[English](../en/10_deploy_deployments.md) | [繁體中文](../zh-tw/10_deploy_deployments.md) | [日本語](../ja/10_deploy_deployments.md) | [回到索引](../README.md)

## 前置準備

#### 啟動 Docker 服務
```bash
$ sudo service docker start
$ docker ps
```

#### 建立 Minikube 叢集
```bash
$ minikube start --driver docker
$ minikube status
```

## 建立 Deployment

#### 建立 Deployment 定義檔
建立 `simple-deployment.yaml` 檔案：

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
        ports: 
        - containerPort: 80
```

#### 部署 Deployment
```bash
# 部署 Deployment
$ kubectl apply -f simple-deployment.yaml

# 查看 Deployment 狀態
$ kubectl get deployments
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
app-deployment   3/3     3            3           19s

# 查看 ReplicaSet 狀態
$ kubectl get rs
NAME                        DESIRED   CURRENT   READY   AGE
app-deployment-8687d78656   3         3         3       28s

# 查看 Pod 狀態
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-8687d78656-5bf82   1/1     Running   0          31s
app-deployment-8687d78656-l4ctt   1/1     Running   0          31s
app-deployment-8687d78656-lv7wl   1/1     Running   0          31s

# 查看所有資源
$ kubectl get all
AME                                  READY   STATUS    RESTARTS   AGE
pod/app-deployment-8687d78656-5bf82   1/1     Running   0          37s
pod/app-deployment-8687d78656-l4ctt   1/1     Running   0          37s
pod/app-deployment-8687d78656-lv7wl   1/1     Running   0          37s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   51m

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/app-deployment   3/3     3            3           37s

NAME                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/app-deployment-8687d78656   3         3         3       37s
```

## 測試 Deployment 狀態維持機制

#### 模擬 Pod 故障
```bash
# 刪除所有 Pod，測試 Deployment 的自動重建功能
$ kubectl delete pods --all
pod "app-deployment-8687d78656-5bf82" deleted
pod "app-deployment-8687d78656-l4ctt" deleted
pod "app-deployment-8687d78656-lv7wl" deleted

# 觀察 Deployment 自動建立新的 Pod
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-8687d78656-2hhtc   1/1     Running   0          37s
app-deployment-8687d78656-4hcvv   1/1     Running   0          37s
app-deployment-8687d78656-4xfcd   1/1     Running   0          37s
```

#### 測試 ReplicaSet 刪除
```bash
# 刪除所有 ReplicaSet
$ kubectl delete rs --all
replicaset.apps "app-deployment-8687d78656" deleted

# 觀察 Deployment 自動建立新的 ReplicaSet
$ kubectl get rs
NAME                        DESIRED   CURRENT   READY   AGE
app-deployment-8687d78656   3         3         3       32s

# 確認 Pod 仍然存在
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-8687d78656-6hzmm   1/1     Running   0          78s
app-deployment-8687d78656-865xb   1/1     Running   0          78s
app-deployment-8687d78656-qm75q   1/1     Running   0          78s
```

## 清理資源
```bash
# 刪除所有 Deployment
$ kubectl delete deployments --all
```

## 說明

- **Deployment**: 提供 Pod 和 ReplicaSet 的宣告式更新，是 Kubernetes 中最常用的工作負載資源
- **replicas: 3**: 指定要維持的 Pod 副本數量
- **selector**: 定義如何識別屬於此 Deployment 的 Pod
- **template**: 定義 Pod 的規格，包含容器映像、端口等設定
- **自我修復**: 當 Pod 被刪除或故障時，Deployment 會自動建立新的 Pod 來維持指定的副本數量
- **kubectl get deployments**: 查看 Deployment 狀態，顯示準備就緒的副本數、最新版本副本數等資訊

## 重要概念

- **Deployment vs ReplicaSet**: Deployment 管理 ReplicaSet，而 ReplicaSet 管理 Pod，形成層級化的管理結構
- **期望狀態 vs 實際狀態**: Deployment 會持續監控並確保實際運行的 Pod 數量符合期望的副本數
- **標籤選擇器**: 使用 `matchLabels` 來識別和管理屬於此 Deployment 的 Pod
- **Pod 模板**: 定義新建立的 Pod 應該具備的規格和設定
- **版本控制**: Deployment 支援滾動更新和版本回滾功能（進階功能）
- **資源層級**: Deployment > ReplicaSet > Pod，刪除上層資源會影響下層資源
