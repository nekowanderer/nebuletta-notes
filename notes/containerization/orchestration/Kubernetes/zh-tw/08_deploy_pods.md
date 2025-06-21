# Kubernetes Pod 部署實作

[English](../en/08_deploy_pods.md) | [繁體中文](../zh-tw/08_deploy_pods.md) | [日本語](../ja/08_deploy_pods.md) | [回到索引](../README.md)

## 前置準備

#### 啟動 Docker 服務
```bash
sudo service docker start
docker ps
```

#### 建立 Minikube 叢集
```bash
minikube start --driver docker
minikube status
```

## 建立 Pod

#### 建立 Pod 定義檔
建立 `simple-pod.yaml` 檔案：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app-container
    image: uopsdod/k8s-hostname-amd64-beta:v1
    ports:
    - containerPort: 80
```

#### 部署 Pod
```bash
# 部署 Pod
kubectl apply -f simple-pod.yaml

# 查看 Pod 狀態
kubectl get pods

# 即時監控 Pod 狀態變化
kubectl get pods -w
```

## 清理資源
```bash
# 刪除所有 Pod
kubectl delete pods --all
```

## 說明

- **Pod**: Kubernetes 中最小的部署單位，可以包含一個或多個容器
- **kubectl apply**: 建立或更新 Kubernetes 資源
- **kubectl get pods -w**: 或是 `--watch`，即時監控 Pod 狀態，按 Ctrl+C 停止監控
- **kubectl delete pods --all**: 刪除所有 Pod 進行清理
