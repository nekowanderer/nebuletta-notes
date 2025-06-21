# Kubernetes ReplicaSet 部署實作

[English](../en/09_deploy_replica_set.md) | [繁體中文](../zh-tw/09_deploy_replica_set.md) | [日本語](../ja/09_deploy_replica_set.md) | [回到索引](../README.md)

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

## 建立 ReplicaSet

#### 建立 ReplicaSet 定義檔
建立 `simple-replicaset.yaml` 檔案：

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: app-rs
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

#### 部署 ReplicaSet
```bash
# 部署 ReplicaSet
$ kubectl apply -f simple-replicaset.yaml

# 查看 ReplicaSet 狀態
$ kubectl get rs
NAME     DESIRED   CURRENT   READY   AGE
app-rs   3         3         3       26s

# 查看 Pod 狀態，等待 Pod 啟動完成
$ kubectl get pods
NAME           READY   STATUS    RESTARTS   AGE
app-rs-24r5n   1/1     Running   0          32s
app-rs-7fswq   1/1     Running   0          32s
app-rs-bhc7x   1/1     Running   0          32s
```

## 測試 ReplicaSet 自我修復機制

#### 模擬 Pod 故障
```bash
# 刪除所有 Pod，測試 ReplicaSet 的自動重建功能
$ kubectl delete pods --all
pod "app-rs-24r5n" deleted
pod "app-rs-7fswq" deleted
pod "app-rs-bhc7x" deleted

# 觀察 ReplicaSet 自動建立新的 Pod
$ kubectl get pods
NAME           READY   STATUS    RESTARTS   AGE
app-rs-4mb4x   1/1     Running   0          35s
app-rs-rjdvn   1/1     Running   0          35s
app-rs-vkct7   1/1     Running   0          35s
```

## 清理資源
```bash
# 刪除所有 ReplicaSet
$ kubectl delete rs --all
```

## 說明

- **ReplicaSet**: 確保指定數量的 Pod 副本持續運行，提供高可用性和負載分散
- **replicas: 3**: 指定要維持的 Pod 副本數量
- **selector**: 定義如何識別屬於此 ReplicaSet 的 Pod
- **template**: 定義 Pod 的規格，包含容器映像、端口等設定
- **自我修復**: 當 Pod 被刪除或故障時，ReplicaSet 會自動建立新的 Pod 來維持指定的副本數量
- **kubectl get rs**: 查看 ReplicaSet 狀態，顯示期望副本數、實際副本數等資訊

## 重要概念

- **期望狀態 vs 實際狀態**: ReplicaSet 會持續監控並確保實際運行的 Pod 數量符合期望的副本數
- **標籤選擇器**: 使用 `matchLabels` 來識別和管理屬於此 ReplicaSet 的 Pod
- **Pod 模板**: 定義新建立的 Pod 應該具備的規格和設定 
