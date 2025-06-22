# Kubernetes Service ClusterIP 實作指南

[English](../en/13_k8s_clusterip_mode.md) | [繁體中文](../zh-tw/13_k8s_clusterip_mode.md) | [日本語](../ja/13_k8s_clusterip_mode.md) | [回到索引](../README.md)

## 概述

ClusterIP 是 Kubernetes 中最基本的 Service 類型，主要用於叢集內部 Pod 之間的通信。

## 前置準備

### 1. 啟動 Docker 環境
```bash
# 啟動 Docker 服務
$ sudo service docker start

# 確認 Docker 正在運行
$ docker ps
```

### 2. 建立 Minikube 叢集
```bash
# 使用 Docker driver 啟動 Minikube
$ minikube start --driver docker

# 檢查叢集狀態
$ minikube status
```

## 部署應用程式

重複利用 [10_deploy_deployments](./10_deploy_deployments.md) 當中的 `simple-deployment.yaml`

### 1. 部署 Deployment
```bash
# 部署應用程式
$ kubectl apply -f simple-deployment.yaml

# 檢查 Deployment 狀態
$ kubectl get deployments
```

## 建立 ClusterIP Service

### 1. 建立 Service 設定檔
建立 `simple-service-clusterip.yaml` 檔案：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service-clusterip
spec:
  type: ClusterIP
  selector:
    app: app-pod
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
```

### 2. 部署 Service
```bash
# 部署 ClusterIP Service
$ kubectl apply -f simple-service-clusterip.yaml

# 查看所有 Service
$ kubectl get services
NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
app-service-clusterip   ClusterIP   10.105.138.111   <none>        8080/TCP   7s
kubernetes              ClusterIP   10.96.0.1        <none>        443/TCP    25h

# 查看 Service 詳細資訊
$ kubectl describe service app-service-clusterip
ame:                     app-service-clusterip
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=app-pod
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.105.138.111
IPs:                      10.105.138.111
Port:                     <unset>  8080/TCP
TargetPort:               80/TCP
Endpoints:                10.244.0.22:80,10.244.0.21:80,10.244.0.20:80
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>
```

## 測試 ClusterIP 連線

### 1. 取得 Service IP
```bash
# 查看 Service 資訊，取得 ClusterIP
$ kubectl get services
```

### 2. 進入 Minikube 容器測試
```bash
# 查看 Docker 容器
$ docker ps

# 進入 Minikube 容器
$ docker exec -it minikube bash

# 使用 Service IP 測試連線（替換 [service_ip] 為實際的 ClusterIP）
$ curl [service_ip]:8080

# 退出容器
$ exit
```

## 重要概念說明

### ClusterIP 特性
- **叢集內部通訊**：只能在 Kubernetes 叢集內部存取
- **自動負載平衡**：kube-proxy 會自動將流量分配到後端 Pod
- **DNS 解析**：叢集內可使用 Service 名稱進行 DNS 解析
- **IP 固定**：Service IP 在叢集內保持固定，即使 Pod 重啟

### 網路流程
```
Pod A → ClusterIP Service → Pod B
```

### 適用場景
- 微服務之間的內部通訊
- 資料庫連線
- 內部 API 呼叫
- 監控和日誌收集服務

## 故障排除

### 常見問題
1. **Service 無法連線**
   - 檢查 Pod 標籤是否與 Service selector 匹配
   - 確認 Pod 正在運行且健康

2. **Port 不匹配**
   - 確認 Service 的 `targetPort` 與 Pod 的容器 Port 一致

3. **DNS 解析問題**
   - 在叢集內使用完整 DNS 名稱：`service-name.namespace.svc.cluster.local`

## 相關資源

- [Kubernetes Service 官方文件](https://kubernetes.io/docs/concepts/services-networking/service/)
- [ClusterIP Service 類型說明](../11_k8s_service_types.md)
