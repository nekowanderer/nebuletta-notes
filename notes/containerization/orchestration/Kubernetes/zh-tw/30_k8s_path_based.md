# Kubernetes Path-based Ingress 實作指南

[English](../en/30_k8s_path_based.md) | [繁體中文](../zh-tw/30_k8s_path_based.md) | [日本語](../ja/30_k8s_path_based.md) | [回到索引](../README.md)

## 概述

Path-based Ingress 是 Kubernetes 中一種重要的流量路由方式，允許根據 URL 路徑將請求轉發到不同的後端服務。本指南將展示如何在 Minikube 環境中實作 Path-based Ingress，將 `/beta` 和 `/prod` 路徑分別路由到不同的應用程式服務。

## 環境準備

### 啟動 Docker 服務

```bash
$ sudo service docker start
$ docker ps
```

### 建立 Minikube 叢集

```bash
$ minikube start --driver docker
$ minikube status
```

### 啟用 Ingress 附加元件

```bash
$ minikube addons enable ingress
$ kubectl get all -n ingress-nginx
```

## 應用程式部署

### 部署 Beta 應用程式

```bash
$ kubectl apply -f beta-app-all.yaml
$ kubectl get all -n app-ns
```

### 部署 Production 應用程式

```bash
$ kubectl apply -f prod-app-all.yaml
$ kubectl get all -n app-ns
```

## Ingress 配置

### Path-based Ingress 設定檔

建立 `ingress-path.yaml` 檔案：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-path
  namespace: app-ns
spec:
  ingressClassName: nginx
  rules:
  - host: all.demo.com
    http:
      paths:
      - pathType: Prefix
        path: /beta
        backend:
          service:
            name: beta-app-service-clusterip
            port:
              number: 8080
      - pathType: Prefix
        path: /prod
        backend:
          service:
            name: prod-app-service-clusterip
            port:
              number: 8080
```

### 部署 Ingress

```bash
$ kubectl apply -f ingress-path.yaml

$ kubectl get ingress -n app-ns
NAME               CLASS   HOSTS                         ADDRESS        PORTS   AGE
ingress-hostname   nginx   beta.demo.com,prod.demo.com   192.168.49.2   80      26m
ingress-path       nginx   all.demo.com,prod.demo.com    192.168.49.2   80      63s
```

## 配置說明

### Path-based 路由規則

- **`/beta` 路徑**：轉發到 `beta-app-service-clusterip` 服務
- **`/prod` 路徑**：轉發到 `prod-app-service-clusterip` 服務
- **`pathType: Prefix`**：使用前綴匹配，`/beta` 會匹配 `/beta`、`/beta/`、`/beta/api` 等路徑

### 關鍵元件

- **`ingressClassName: nginx`**：指定使用 NGINX Ingress Controller
- **`host: all.demo.com`**：設定虛擬主機名稱
- **`pathType: Prefix`**：路徑匹配類型為前綴匹配

## 測試驗證

### 1. 取得 Ingress IP

```bash
$ kubectl get ingress -n app-ns
```

### 2. 測試 Beta 應用程式

```bash
$ curl [ingress_ip]:80/beta -H 'Host: all.demo.com'
```

### 3. 測試 Production 應用程式

```bash
$ curl [ingress_ip]:80/prod -H 'Host: all.demo.com'
```

## 資源清理

完成測試後，清理所有資源：

```bash
$ kubectl delete ns app-ns
```

## 注意事項

1. **路徑優先順序**：Ingress 控制器會按照配置順序匹配路徑，較長的路徑應該放在較短的路徑之前
2. **Host 標頭**：測試時必須包含正確的 Host 標頭，否則請求可能無法正確路由
3. **命名空間**：確保所有資源都在同一個命名空間中，或正確指定命名空間
4. **服務可用性**：部署 Ingress 前確保後端服務已正常運行

## 常見問題

Q: 為什麼需要設定 Host 標頭？
A: Ingress 規則中設定了 `host: all.demo.com`，因此測試時必須包含此 Host 標頭才能正確匹配規則。

Q: 如何新增更多路徑？
A: 在 `paths` 陣列中新增新的路徑配置，指定 `path`、`pathType` 和對應的後端服務。

Q: 如何設定不同的路徑類型？
A: `pathType` 可以設定為：
- `Prefix`：前綴匹配
- `Exact`：精確匹配
- `ImplementationSpecific`：實作特定匹配
