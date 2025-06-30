# Kubernetes Ingress Hostname Based 實作指南

[English](../en/29_k8s_ingress_hostname_based.md) | [繁體中文](../zh-tw/29_k8s_ingress_hostname_based.md) | [日本語](../ja/29_k8s_ingress_hostname_based.md) | [回到索引](../README.md)

## 概述

本筆記說明如何在 Kubernetes 中使用 Ingress 控制器實現基於主機名（Hostname-based）的路由，讓不同的域名指向不同的服務。

## 前置準備

### 1. 啟動 Docker 和 Minikube

```bash
# 啟動 Docker 服務
$ sudo service docker start
$ docker ps

# 建立 Minikube 叢集
$ minikube start --driver docker
$ minikube status
```

### 2. 啟用 Ingress 附加元件

```bash
# 啟用 minikube ingress 附加元件
$ minikube addons enable ingress
$ kubectl get all -n ingress-nginx
NAME                                           READY   STATUS      RESTARTS      AGE
pod/ingress-nginx-admission-create-5fj8c       0/1     Completed   0             22h
pod/ingress-nginx-admission-patch-529hj        0/1     Completed   1             22h
pod/ingress-nginx-controller-67c5cb88f-dv4px   1/1     Running     1 (14m ago)   22h

NAME                                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx-controller             NodePort    10.102.245.181   <none>        80:30447/TCP,443:31545/TCP   22h
service/ingress-nginx-controller-admission   ClusterIP   10.96.249.142    <none>        443/TCP                      22h

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-nginx-controller   1/1     1            1           22h

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-nginx-controller-67c5cb88f   1         1         1       22h

NAME                                       STATUS     COMPLETIONS   DURATION   AGE
job.batch/ingress-nginx-admission-create   Complete   1/1           7s         22h
job.batch/ingress-nginx-admission-patch    Complete   1/1           7s         22h
```

## 部署應用程式

### 1. 部署 Beta 環境應用程式

```bash
# 部署 beta 環境的所有資源
$ kubectl apply -f beta-app-all.yaml
$ kubectl get all -n app-ns
NAME                                       READY   STATUS    RESTARTS   AGE
pod/beta-app-deployment-865c646d9d-7lvf2   1/1     Running   0          2m35s
pod/beta-app-deployment-865c646d9d-ffpzt   1/1     Running   0          2m35s
pod/beta-app-deployment-865c646d9d-npjt2   1/1     Running   0          2m35s
pod/prod-app-deployment-586c5dcc59-dctfp   1/1     Running   0          37s
pod/prod-app-deployment-586c5dcc59-dhpr7   1/1     Running   0          37s
pod/prod-app-deployment-586c5dcc59-v9hm7   1/1     Running   0          37s

NAME                                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/beta-app-service-clusterip   ClusterIP   10.105.100.5     <none>        8080/TCP   2m35s
service/prod-app-service-clusterip   ClusterIP   10.108.143.220   <none>        8080/TCP   37s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/beta-app-deployment   3/3     3            3           2m35s
deployment.apps/prod-app-deployment   3/3     3            3           37s

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/beta-app-deployment-865c646d9d   3         3         3       2m35s
replicaset.apps/prod-app-deployment-586c5dcc59   3         3         3       37s
```

### 2. 部署 Production 環境應用程式

首先複製並修改配置檔案：

```bash
# 複製 beta 配置為 prod 配置
$ cp beta-app-all.yaml prod-app-all.yaml
$ vi prod-app-all.yaml
```

**prod-app-all.yaml 內容：**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app-ns
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prod-app-deployment
  namespace: app-ns
spec:
  replicas: 3
  selector:
    matchLabels:
      app: prod-app-pod
  template:
    metadata:
      labels:
        app: prod-app-pod
    spec:
      containers:
      - name: prod-app-container
        image: uopsdod/k8s-hostname-amd64-prod:v1
        ports: 
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: prod-app-service-clusterip
  namespace: app-ns
spec:
  selector:
    app: prod-app-pod
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
```

部署 production 環境：

```bash
# 部署 prod 環境的所有資源
$ kubectl apply -f prod-app-all.yaml
$ kubectl get all -n app-ns
```

## 配置 Ingress 路由

### 建立 Ingress 配置檔案

**ingress-hostname.yaml 內容：**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-hostname
  namespace: app-ns
spec:
  ingressClassName: nginx
  rules:
  - host: beta.demo.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: beta-app-service-clusterip
            port:
              number: 8080
  - host: prod.demo.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: prod-app-service-clusterip
            port:
              number: 8080
```

### 部署 Ingress

```bash
# 部署 Ingress 配置
$ kubectl apply -f ingress-hostname.yaml
ingress.networking.k8s.io/ingress-hostname created

$ kubectl get all -n app-ns

$ kubectl get ingress -n app-ns
NAME               CLASS   HOSTS                         ADDRESS        PORTS   AGE
ingress-hostname   nginx   beta.demo.com,prod.demo.com   192.168.49.2   80      93s
```

## 測試驗證

### 測試連線

使用 curl 命令測試不同主機名的路由：

```bash
# 測試 beta 環境
$ curl [ingress_ip]:80 -H 'Host: beta.demo.com'
$ curl [ingress_ip]:80 -H 'Host: beta.demo.com'
$ curl [ingress_ip]:80 -H 'Host: beta.demo.com'

# 測試 production 環境
$ curl [ingress_ip]:80 -H 'Host: prod.demo.com'
$ curl [ingress_ip]:80 -H 'Host: prod.demo.com'
$ curl [ingress_ip]:80 -H 'Host: prod.demo.com'
```

## 架構說明

### 路由機制

- **beta.demo.com** → 路由到 `beta-app-service-clusterip`
- **prod.demo.com** → 路由到 `prod-app-service-clusterip`

### 關鍵元件

1. **Ingress Controller**: 使用 nginx 作為 Ingress 控制器
2. **Host Rules**: 基於 HTTP Host 標頭進行路由判斷
3. **Service Backend**: 每個主機名對應到不同的 ClusterIP Service
