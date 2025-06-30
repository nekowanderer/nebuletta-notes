# Kubernetes Ingress 總結

[English](../en/31_k8s_ingress_summary.md) | [繁體中文](../zh-tw/31_k8s_ingress_summary.md) | [日本語](../ja/31_k8s_ingress_summary.md) | [回到索引](../README.md)

## 概述

本筆記總結 Kubernetes Ingress 的三種主要組態方式：Default Backend、Hostname-based 和 Path-based。每種方式都有其特定的適用場景和配置方法。

## 三種組態方式比較

### 1. Default Backend（預設後端）

#### 適用情境
- **404 錯誤處理**：當請求不匹配任何規則時提供預設回應
- **維護頁面**：在系統維護期間顯示統一的維護頁面
- **錯誤處理**：提供統一的錯誤頁面或 API 回應
- **簡單應用**：只有單一服務需要對外暴露

#### YAML 配置範例
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
      name: default-app-service
      port:
        number: 8080
```

#### 特點
- **最簡單的配置**：只需要指定 `defaultBackend`
- **全域預設**：所有未匹配的請求都會導向預設服務
- **無路由規則**：不包含 `rules` 區段
- **單一入口點**：適合單一應用程式或簡單架構

### 2. Hostname-based（主機名路由）

#### 適用情境
- **多租戶架構**：不同客戶使用不同域名
- **環境分離**：beta.demo.com、prod.demo.com 等
- **微服務架構**：每個服務有獨立的域名
- **品牌分離**：不同品牌使用不同域名

#### YAML 配置範例
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
            name: beta-app-service
            port:
              number: 8080
  - host: prod.demo.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: prod-app-service
            port:
              number: 8080
```

#### 特點
- **域名分離**：不同域名指向不同服務
- **清晰的分離**：每個環境或服務有獨立的域名
- **易於管理**：可以為每個域名設定不同的 SSL 憑證
- **適合多環境**：開發、測試、正式環境分離

### 3. Path-based（路徑路由）

#### 適用情境
- **單一域名多服務**：在同一個域名下提供多個服務
- **API 網關**：統一入口點，按路徑路由到不同 API
- **前端路由**：SPA 應用程式的後端 API 路由
- **版本控制**：不同版本的 API 使用不同路徑

#### YAML 配置範例
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-path
  namespace: app-ns
spec:
  ingressClassName: nginx
  rules:
  - host: api.demo.com
    http:
      paths:
      - pathType: Prefix
        path: /beta
        backend:
          service:
            name: beta-app-service
            port:
              number: 8080
      - pathType: Prefix
        path: /prod
        backend:
          service:
            name: prod-app-service
            port:
              number: 8080
      - pathType: Prefix
        path: /api/v1
        backend:
          service:
            name: api-v1-service
            port:
              number: 8080
```

#### 特點
- **路徑分離**：同一域名下按路徑路由
- **統一入口**：所有服務共用同一個域名
- **靈活配置**：可以組合多種路徑類型
- **適合 API 設計**：RESTful API 的標準做法

## 配置差異比較

### YAML 結構差異

| 組態方式 | 主要區段 | 路由規則 | 後端配置 |
|---------|---------|---------|---------|
| Default Backend | `defaultBackend` | 無 | 單一服務 |
| Hostname-based | `rules[].host` | 按域名 | 每個域名一個服務 |
| Path-based | `rules[].http.paths` | 按路徑 | 每個路徑一個服務 |

### 路由優先順序

1. **Hostname-based**：優先匹配域名
2. **Path-based**：在相同域名下按路徑匹配
3. **Default Backend**：當所有規則都不匹配時使用

### 測試方法差異

#### Default Backend
```bash
$ curl [ingress_ip]:80
```

#### Hostname-based
```bash
$ curl [ingress_ip]:80 -H 'Host: beta.demo.com'
$ curl [ingress_ip]:80 -H 'Host: prod.demo.com'
```

#### Path-based
```bash
$ curl [ingress_ip]:80/beta -H 'Host: api.demo.com'
$ curl [ingress_ip]:80/prod -H 'Host: api.demo.com'
```

## 選擇建議

### 選擇 Default Backend 當：
- 只有單一服務需要對外暴露
- 需要統一的錯誤處理機制
- 系統架構簡單，不需要複雜路由

### 選擇 Hostname-based 當：
- 需要為不同環境或客戶提供獨立域名
- 需要為不同域名設定不同的 SSL 憑證
- 希望有清晰的服務分離

### 選擇 Path-based 當：
- 希望使用單一域名提供多個服務
- 設計 RESTful API 架構
- 需要版本控制或環境分離但希望共用域名

### 組合使用
在實際應用中，這三種方式可以組合使用：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-combined
  namespace: app-ns
spec:
  ingressClassName: nginx
  defaultBackend:
    service:
      name: error-page-service
      port:
        number: 8080
  rules:
  - host: api.demo.com
    http:
      paths:
      - pathType: Prefix
        path: /v1
        backend:
          service:
            name: api-v1-service
            port:
              number: 8080
      - pathType: Prefix
        path: /v2
        backend:
          service:
            name: api-v2-service
            port:
              number: 8080
  - host: admin.demo.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: admin-service
            port:
              number: 8080
```

## 注意事項

1. **路徑優先順序**：在 Path-based 配置中，較長的路徑應該放在較短的路徑之前
2. **Host 標頭**：測試時必須包含正確的 Host 標頭
3. **SSL 憑證**：Hostname-based 可以為每個域名設定不同的 SSL 憑證
4. **命名空間**：確保所有相關資源都在正確的命名空間中
5. **服務可用性**：部署 Ingress 前確保後端服務已正常運行

## 總結

Kubernetes Ingress 的三種組態方式各有其適用場景：

- **Default Backend**：適合簡單架構和錯誤處理
- **Hostname-based**：適合多租戶和多環境分離
- **Path-based**：適合 API 網關和統一入口設計

在實際應用中，可以根據需求組合使用這些方式，建立靈活且可擴展的流量路由架構。


