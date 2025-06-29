# Kubernetes Ingress Default Backend 實作指南

[English](../en/28_k8s_ingress_default_backend.md) | [繁體中文](../zh-tw/28_k8s_ingress_default_backend.md) | [日本語](../ja/28_k8s_ingress_default_backend.md) | [回到索引](../README.md)

## 概述
本筆記說明如何在 Kubernetes 中設定 Ingress 的 defaultBackend，當請求的路徑不匹配任何規則時，會將流量導向預設的後端服務。

## 前置準備

### 1. 啟動 Docker 環境
```bash
$ sudo service docker start
$ docker ps
```

### 2. 建立 Minikube 叢集
```bash
$ minikube start --driver docker
$ minikube status
```

### 3. 啟用 Ingress 附加元件
```bash
$ minikube addons enable ingress
💡  ingress is an addon maintained by Kubernetes. For any concerns contact minikube on GitHub.
You can view the list of minikube maintainers at: https://github.com/kubernetes/minikube/blob/master/OWNERS
    ▪ Using image registry.k8s.io/ingress-nginx/kube-webhook-certgen:v1.5.3
    ▪ Using image registry.k8s.io/ingress-nginx/controller:v1.12.2
    ▪ Using image registry.k8s.io/ingress-nginx/kube-webhook-certgen:v1.5.3
🔎  Verifying ingress addon...
🌟  The 'ingress' addon is enabled

$ kubectl get all -n ingress-nginx
NAME                                           READY   STATUS      RESTARTS   AGE
pod/ingress-nginx-admission-create-5fj8c       0/1     Completed   0          64s
pod/ingress-nginx-admission-patch-529hj        0/1     Completed   1          64s
pod/ingress-nginx-controller-67c5cb88f-dv4px   1/1     Running     0          64s

NAME                                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx-controller             NodePort    10.102.245.181   <none>        80:30447/TCP,443:31545/TCP   64s
service/ingress-nginx-controller-admission   ClusterIP   10.96.249.142    <none>        443/TCP                      64s

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-nginx-controller   1/1     1            1           64s

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-nginx-controller-67c5cb88f   1         1         1       64s

NAME                                       STATUS     COMPLETIONS   DURATION   AGE
job.batch/ingress-nginx-admission-create   Complete   1/1           7s         64s
job.batch/ingress-nginx-admission-patch    Complete   1/1           7s         64s
```

## 部署應用程式

### 1. 部署測試應用程式
```bash
$ kubectl apply -f beta-app-all.yaml
namespace/app-ns created
deployment.apps/beta-app-deployment created
service/beta-app-service-clusterip created

$ kubectl get all -n app-ns
NAME                                       READY   STATUS    RESTARTS   AGE
pod/beta-app-deployment-865c646d9d-6f8sj   1/1     Running   0          31s
pod/beta-app-deployment-865c646d9d-c9ltr   1/1     Running   0          31s
pod/beta-app-deployment-865c646d9d-zfk9f   1/1     Running   0          31s

NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/beta-app-service-clusterip   ClusterIP   10.108.189.68   <none>        8080/TCP   31s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/beta-app-deployment   3/3     3            3           31s

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/beta-app-deployment-865c646d9d   3         3         3       31s
```

## 設定 Ingress Default Backend

### 1. 建立 Ingress 設定檔
建立 `ingress-defaultbackend.yaml` 檔案：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-defaultbackend
  namespace: app-ns
spec:
  ingressClassName: nginx
  defaultBackend:
    service:
      name: beta-app-service-clusterip
      port:
        number: 8080
```

### 2. 部署 Ingress
```bash
$ kubectl apply -f ingress-defaultbackend.yaml
ingress.networking.k8s.io/ingress-defaultbackend created

$ kubectl get ingress -n app-ns
NAME                     CLASS   HOSTS   ADDRESS        PORTS   AGE
ingress-defaultbackend   nginx   *       192.168.49.2   80      17s

$ kubectl describe ingress ingress-defaultbackend -n app-ns
Name:             ingress-defaultbackend
Labels:           <none>
Namespace:        app-ns
Address:          192.168.49.2
Ingress Class:    nginx
Default backend:  beta-app-service-clusterip:8080 (10.244.0.68:80,10.244.0.67:80,10.244.0.69:80)
Rules:
  Host        Path  Backends
  ----        ----  --------
  *           *     beta-app-service-clusterip:8080 (10.244.0.68:80,10.244.0.67:80,10.244.0.69:80)
Annotations:  <none>
Events:
  Type    Reason  Age                From                      Message
  ----    ------  ----               ----                      -------
  Normal  Sync    41s (x2 over 43s)  nginx-ingress-controller  Scheduled for sync

$ kubectl get ingress -n app-ns -w
```

## 測試驗證

### 1. 取得 Ingress IP
```bash
$ kubectl get ingress -n app-ns
```

### 2. 測試連線
```bash
$ curl [ingress_ip]:80
$ curl [ingress_ip]:80
$ curl [ingress_ip]:80
$ curl [ingress_ip]:80
$ curl [ingress_ip]:80
```

## 重要說明

- **defaultBackend 功能**：當請求的路徑不匹配任何 Ingress 規則時，流量會被導向指定的預設後端服務
- **適用場景**：適用於需要處理 404 錯誤或提供預設回應的應用程式
- **注意事項**：確保指定的後端服務存在且正常運作

## 清理資源
```bash
$ kubectl delete -f ingress-defaultbackend.yaml

$ kubectl delete -f beta-app-all.yaml

$ minikube stop
```

