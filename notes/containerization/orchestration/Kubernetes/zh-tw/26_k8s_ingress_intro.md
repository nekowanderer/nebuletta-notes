# Kubernetes Ingress 入門介紹

[English](../en/26_k8s_ingress_intro.md) | [繁體中文](../zh-tw/26_k8s_ingress_intro.md) | [日本語](../ja/26_k8s_ingress_intro.md) | [回到索引](../README.md)

```
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                                Cluster                                  │
  │  ┌─────────────┐                                                        │
  │  │  Service A  │                                                        │
  │  │ ┌───┐ ┌───┐ │    all.demo.com/beta                                   │
  │  │ │Pod│ │Pod│ │ ←─────┐                                                │
  │  │ └───┘ └───┘ │       │                                                │
  │  └─────────────┘       │                                                │
  │                        │                                                │
  │                        │    ┌─────────┐   ┌────────────────────┐        │
  │                        └─── │         │   │                    │        │
  │                             │ Ingress │ ← │ Ingress Controller │        │
  │                        ┌─── │         │   │                    │        │
  │                        │    └─────────┘   └────────────────────┘        │
  │                        │                           ↑                    │
  │  ┌─────────────┐       │                           │                    │
  │  │  Service B  │       │                  ┌─────────────────┐           │
  │  │ ┌───┐ ┌───┐ │ ←─────┘                  │  Load Balancer  │ ←── User  │
  │  │ │Pod│ │Pod│ │    all.demo.com/prod     └─────────────────┘           │
  │  │ └───┘ └───┘ │                                                        │
  │  └─────────────┘                                                        │
  └─────────────────────────────────────────────────────────────────────────┘
```

### 什麼是 Kubernetes Ingress

Kubernetes Ingress 是一個 API 物件，用來管理叢集內服務的外部存取。它作為 HTTP 和 HTTPS 流量的單一進入點，提供規則來根據主機名、路徑或其他條件將流量路由到不同的服務。

### 基本 Ingress 架構

基本的 Ingress 設定包含幾個關鍵元件：

- **Ingress 資源**：定義路由規則
- **Ingress Controller**：實作實際的路由邏輯
- **負載平衡器**：流量的外部進入點
- **服務**：接收路由流量的後端服務

```
外部使用者 → 負載平衡器 → Ingress Controller → Ingress → 服務 → Pod
```

### 單一服務路由與預設後端 (default backend)

在最簡單的配置中，Ingress 可以將所有流量導向單一服務：

- 外部請求透過負載平衡器進入
- Ingress Controller 處理請求
- 所有流量被路由到**預設後端**服務
- 服務接著將流量轉發到其關聯的 Pod（Service A 包含多個 Pod）

這種設定適用於不需要複雜路由規則的簡單應用程式。

### 基於主機的多服務路由

Ingress 能夠根據不同的 hostname 路由到不同的服務：

- **all.demo.com/beta** 路由到 Service A
- **all.demo.com/prod** 路由到 Service B

每個服務維護自己的 Pod 集合，允許：
- 環境分離（測試版 vs 正式版）
- 服務的獨立擴展
- 在相同網域內的路徑基礎路由

### Ingress Controller 選項

**雲端提供商解決方案：**
- **AWS ALB**（Application Load Balancer）
- **GCP Load Balancer** 
- **Azure Application Gateway**

**自我管理解決方案：**
- **Local Nginx** - 最常見的開源選項

Ingress Controller 的選擇取決於：
- 基礎設施平台（雲端 vs 地端）
- 功能需求（SSL 終止、身份驗證等）
- 效能和成本考量

### 使用 Ingress 的主要優勢

- **單一進入點**：透過一個介面整合外部存取
- **成本效益**：減少對多個 LoadBalancer 服務的需求
- **進階路由**：支援基於主機和路徑的路由
- **SSL 終止**：可以集中處理 HTTPS 憑證
- **負載分散**：支援各種負載平衡演算法