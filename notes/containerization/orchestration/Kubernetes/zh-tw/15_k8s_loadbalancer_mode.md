# Kubernetes Service LoadBalancer 實作指南

[English](../en/15_k8s_loadbalancer_mode.md) | [繁體中文](../zh-tw/15_k8s_loadbalancer_mode.md) | [日本語](../ja/15_k8s_loadbalancer_mode.md) | [回到索引](../README.md)

## 概述
LoadBalancer 類型的 Service 是 Kubernetes 中最常用的外部存取方式之一，它會自動建立一個外部負載平衡器來分配流量到後端的 Pod。

## 前置準備

##### 1. 啟動 Docker 服務
```bash
$ sudo service docker start
$ docker ps
```

##### 2. 建立 Minikube 叢集
```bash
$ minikube start --driver docker
$ minikube status
```

## 部署應用程式

重複利用 [10_deploy_deployments](./10_deploy_deployments.md) 當中的 `simple-deployment.yaml`

##### 1. 部署 Deployment
```bash
$ kubectl apply -f simple-deployment.yaml
$ kubectl get deployments
```

##### 2. 建立 LoadBalancer Service

建立 `simple-service-loadbalancer.yaml` 檔案：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: app-pod
  ports:
    - protocol: TCP
      port: 8080        # Service 對外開放的埠號
      targetPort: 80    # Pod 內部的埠號
```

##### 3. 部署 Service
```bash
$ kubectl apply -f simple-service-loadbalancer.yaml

$ kubectl get services
NAME                       TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
app-service-loadbalancer   LoadBalancer   10.110.110.138   <pending>     8080:31830/TCP   6s  # 注意 EXTERNAL-IP 的部分是 pending
app-service-nodeport       NodePort       10.106.158.207   <none>        8080:30080/TCP   26m
kubernetes                 ClusterIP      10.96.0.1        <none>        443/TCP          26h

$ kubectl describe service app-service-loadbalancer
Name:                     app-service-loadbalancer
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=app-pod
Type:                     LoadBalancer
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.110.110.138
IPs:                      10.110.110.138
Port:                     <unset>  8080/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  31830/TCP
Endpoints:                10.244.0.22:80,10.244.0.21:80,10.244.0.20:80
Session Affinity:         None
External Traffic Policy:  Cluster
Internal Traffic Policy:  Cluster
Events:                   <none>
```

## 啟用外部存取

##### 1. 啟動 Minikube Tunnel
在另一個終端機視窗中執行：
```bash
$ minikube tunnel
Status:
        machine: minikube
        pid: 25727
        route: 10.96.0.0/12 -> 192.168.49.2
        minikube: Running
        services: [app-service-loadbalancer]
    errors:
                minikube: no errors
                router: no errors
                loadbalancer emulator: no errors
```

這個指令會建立一個網路通道，在 local 模擬一個 load balancer，會把流量導向 minikube cluster 之中

##### 2. 測試負載平衡功能
```bash
# 查看服務狀態
$ kubectl get services
NAME                       TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)          AGE
app-service-loadbalancer   LoadBalancer   10.110.110.138   10.110.110.138   8080:31830/TCP   2m23s # EXTERNAL-IP 不是 pending 了，有綁定了
app-service-nodeport       NodePort       10.106.158.207   <none>           8080:30080/TCP   28m
kubernetes                 ClusterIP      10.96.0.1        <none>           443/TCP          26h

# 測試連線（重複執行以觀察負載平衡效果）
$ curl [service_external_ip]:8080
$ curl [service_external_ip]:8080
$ curl [service_external_ip]:8080
$ curl [service_external_ip]:8080
$ curl [service_external_ip]:8080
```

## 資源清理

##### 1. 刪除所有資源
```bash
$ kubectl delete all --all
pod "app-deployment-8687d78656-d6x8t" deleted
pod "app-deployment-8687d78656-gp96l" deleted
pod "app-deployment-8687d78656-p7k6j" deleted
service "app-service-loadbalancer" deleted
service "app-service-nodeport" deleted
service "kubernetes" deleted
deployment.apps "app-deployment" deleted
```

##### 2. 停止 Minikube Tunnel
在執行 `minikube tunnel` 的終端機中按 `Ctrl + C`

## 重要概念

- **LoadBalancer 類型**：自動建立外部負載平衡器
- **port**：Service 對外開放的埠號
- **targetPort**：Pod 內部實際監聽的埠號
- **Minikube Tunnel**：在本地環境中模擬雲端負載平衡器功能

## 注意事項

1. LoadBalancer 類型需要雲端提供者支援
2. 在 Minikube 環境中需要使用 `minikube tunnel` 來模擬外部存取
3. 實際生產環境中，LoadBalancer 會自動分配外部 IP 位址
