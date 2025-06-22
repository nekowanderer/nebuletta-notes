# Kubernetes Service NodePort 實作指南

[English](../en/14_k8s_nodeport_mode.md) | [繁體中文](../zh-tw/14_k8s_nodeport_mode.md) | [日本語](../ja/14_k8s_nodeport_mode.md) | [回到索引](../README.md)

## 概述

NodePort 是 Kubernetes Service 的一種類型，它會在每個節點上開啟一個特定的 Port（範圍 30000-32767），讓外部流量可以透過節點 IP 和該 Port 存取叢集內的服務。

## 前置準備

##### 1. 啟動 Docker 環境
```bash
# 啟動 Docker 服務
$ sudo service docker start

# 確認 Docker 正在運行
$ docker ps
```

##### 2. 建立 Minikube 叢集
```bash
# 使用 Docker driver 啟動 Minikube
$ minikube start --driver docker

# 檢查叢集狀態
$ minikube status
```

## 部署應用程式

重複利用 [10_deploy_deployments](./10_deploy_deployments.md) 當中的 `simple-deployment.yaml`

##### 1. 部署 Deployment
```bash
# 部署應用程式
$ kubectl apply -f simple-deployment.yaml

# 檢查 Deployment 狀態
$ kubectl get deployments
```

## 建立 NodePort Service

##### 1. 建立 Service 設定檔
建立 `simple-service-nodeport.yaml` 檔案：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service-nodeport
spec:
  type: NodePort
  selector:
    app: app-pod
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
      nodePort: 30080  # 可選：指定特定的 NodePort，範圍 30000-32767
```

##### 2. 部署 Service
```bash
# 部署 NodePort Service
$ kubectl apply -f simple-service-nodeport.yaml

# 查看所有 Service
$ kubectl get services
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
app-service-nodeport   NodePort    10.106.158.207   <none>        8080:30080/TCP   5s  # <- 30080 就是 node port
kubernetes             ClusterIP   10.96.0.1        <none>        443/TCP          25h

# 查看 Service 詳細資訊
$ kubectl describe service app-service-nodeport
Name:                     app-service-nodeport
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=app-pod
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.106.158.207
IPs:                      10.106.158.207
Port:                     <unset>  8080/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  30080/TCP
Endpoints:                10.244.0.20:80,10.244.0.22:80,10.244.0.21:80
Session Affinity:         None
External Traffic Policy:  Cluster
Internal Traffic Policy:  Cluster
Events:                   <none>
```

## 取得節點資訊

##### 1. 查看節點資訊
```bash
# 查看所有節點
$ kubectl get nodes
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   25h   v1.33.1

# 查看節點詳細資訊，取得節點 IP
$ kubectl describe node minikube
...
ddresses:
  InternalIP:  192.168.49.2 # <- copy 這個，這個就是 node IP
  Hostname:    minikube
...
```

##### 2. 取得節點 IP
從 `kubectl describe node minikube` 的輸出中，找到 `InternalIP` 或 `ExternalIP` 欄位（此處範例是 `InternalIP`）

## 測試 NodePort 連線

##### 1. 使用節點 IP 和 NodePort 測試
```bash
# 使用節點 IP 和 NodePort 進行測試
# 格式：curl [node_ip]:[nodeport]
$ curl [node_ip]:30080
```

##### 2. 使用 Minikube 提供的服務 URL
```bash
# Minikube 提供便捷的服務 URL 功能
$ minikube service app-service-nodeport --url

# 使用返回的 URL 進行測試
$ curl $(minikube service app-service-nodeport --url)
```

## 重要概念說明

##### NodePort 特性
- **外部存取**：可以從叢集外部透過節點 IP 和 NodePort 存取服務
- **Port 範圍**：NodePort 使用 30000-32767 範圍的 Port
- **節點級別**：每個節點都會開啟相同的 NodePort
- **負載平衡**：kube-proxy 會自動將流量分配到後端 Pod

##### 網路流程
```
外部客戶端 → 節點 IP:NodePort → kube-proxy → Pod
```

##### 適用場景
- 開發環境測試
- 內部網路存取
- 不需要雲端負載平衡器的場景
- 特定節點存取需求

## 進階設定

##### 1. 指定 NodePort
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service-nodeport
spec:
  type: NodePort
  selector:
    app: app-pod
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
      nodePort: 30080  # 指定特定的 NodePort
```

##### 2. External Traffic Policy
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service-nodeport
spec:
  type: NodePort
  externalTrafficPolicy: Local  # 或 Cluster
  selector:
    app: app-pod
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
```

## 故障排除

##### 常見問題
1. **無法從外部連線**
   - 確認防火牆設定允許 NodePort 範圍的流量
   - 檢查節點 IP 是否正確
   - 確認 Pod 正在運行且健康

2. **NodePort 衝突**
   - 檢查是否有其他 Service 使用相同的 NodePort
   - 可以手動指定不同的 NodePort

3. **Port 範圍問題**
   - NodePort 必須在 30000-32767 範圍內
   - 避免使用系統保留的 Port

##### 除錯命令
```bash
# 檢查 Service 狀態
$ kubectl get services

# 檢查 Endpoints
$ kubectl get endpoints app-service-nodeport

# 檢查 Pod 狀態
$ kubectl get pods -l app=app-pod

# 查看 Service 詳細資訊
$ kubectl describe service app-service-nodeport
```

## 相關資源

- [Kubernetes Service 官方文件](https://kubernetes.io/docs/concepts/services-networking/service/)
- [NodePort Service 類型說明](./11_k8s_service_types.md)
- [DNAT 和 SNAT 在 Kubernetes 中的運作](./12_dnat_and_snat_in_k8s.md)
