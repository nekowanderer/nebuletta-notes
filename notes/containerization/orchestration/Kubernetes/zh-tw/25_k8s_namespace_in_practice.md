# Kubernetes Namespace 實務操作

[English](../en/25_k8s_namespace_in_practice.md) | [繁體中文](../zh-tw/25_k8s_namespace_in_practice.md) | [日本語](../ja/25_k8s_namespace_in_practice.md) | [回到索引](../README.md)


## 前置準備

### 啟動 Docker 環境
```bash
# 啟動 Docker 服務
$ sudo service docker start

# 確認 Docker 狀態
$ docker ps
```

### 建立 Minikube 叢集
```bash
# 使用 Docker driver 啟動 minikube
$ minikube start --driver docker

# 檢查叢集狀態
$ minikube status
```

## 建立 Namespace 和應用程式

### 建立 YAML 配置檔案
建立 `beta-app-all.yaml` 檔案：

```yaml
# 建立 namespace
apiVersion: v1
kind: Namespace
metadata:
  name: app-ns
---
# 建立 Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: beta-app-deployment
  namespace: app-ns
spec:
  replicas: 3
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
---
# 建立 Service
apiVersion: v1
kind: Service
metadata:
  name: beta-app-service-clusterip
  namespace: app-ns
spec:
  selector:
    app: beta-app-pod
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
```
其中，`---` 代表在同一份 yaml 檔案之中，我們要進行多個資源種類的部署

## 部署和驗證

### 部署應用程式
```bash
# 部署所有資源
$ kubectl apply -f beta-app-all.yaml
```

### 檢查部署狀態
```bash
# 查看所有 namespace
$ kubectl get ns
NAME              STATUS   AGE
app-ns            Active   7s
default           Active   7d9h
kube-node-lease   Active   7d9h
kube-public       Active   7d9h
kube-system       Active   7d9h

# 查看所有資源（預設 namespace）
$ kubectl get all
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   5d

# 查看指定 namespace 中的所有資源
$ kubectl get all --namespace app-ns
NAME                                       READY   STATUS    RESTARTS   AGE
pod/beta-app-deployment-865c646d9d-9ml47   1/1     Running   0          54s
pod/beta-app-deployment-865c646d9d-shc9b   1/1     Running   0          54s
pod/beta-app-deployment-865c646d9d-xl8v6   1/1     Running   0          54s

NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/beta-app-service-clusterip   ClusterIP   10.103.190.75   <none>        8080/TCP   54s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/beta-app-deployment   3/3     3            3           54s

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/beta-app-deployment-865c646d9d   3         3         3       54s

# 或使用簡短語法
$ kubectl get all -n app-ns
```

## 資源清理

### 刪除 Namespace
```bash
# 刪除整個 namespace（會同時刪除該 namespace 下的所有資源）
$ kubectl delete ns app-ns
namespace "app-ns" deleted

# 確認 namespace 已刪除
$ kubectl get ns
NAME              STATUS   AGE
default           Active   7d9h
kube-node-lease   Active   7d9h
kube-public       Active   7d9h
kube-system       Active   7d9h

# 嘗試查看已刪除的 namespace 中的資源（應該會顯示錯誤）
$ kubectl get all -n app-ns
No resources found in app-ns namespace.
```

## 重要概念說明

- **Namespace 隔離**：不同 namespace 中的資源是相互隔離的
- **資源歸屬**：Deployment 和 Service 都屬於指定的 namespace
- **刪除行為**：刪除 namespace 會同時刪除該 namespace 下的所有資源
- **跨 Namespace 存取**：預設情況下，不同 namespace 的資源無法直接存取
