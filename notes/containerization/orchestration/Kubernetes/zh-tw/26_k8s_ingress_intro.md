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

導入 Ingress 之前（每個服務都有自己的 Load Balancer）：
```
Internet
    |
    v
[Load Balancer A] -> [Service A]
[Load Balancer B] -> [Service B]
[Load Balancer C] -> [Service C]
```

導入 Ingress 之後：
```
External User → Load Balancer → Ingress Controller → Ingress → Service → Pod
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

### Ingress vs Ingress Controller

| 項目 | Ingress Controller（警衛） | Ingress（門禁卡發放規則） | 
|------|---------------------------|---------------------------|
| **本質** | Kubernetes 中運行的程式，通常是 Pod/Deployment | Kubernetes 中的一種資源（Resource） | 
| **功能** | 實際執行 Ingress 規則的組件 | 定義路由規則和策略 |
| **比喻** | 類似於「警衛」，負責檢查門禁卡並執行規則 | 類似於「門禁卡發放的規則」，說明誰可以進入哪些區域 |

#### 兩者關係
- **缺一不可**：Ingress Controller 和 Ingress 必須同時存在才能運作
- **分工明確**：Ingress 定義規則，Controller 執行規則
- **動態協作**：Controller 會監聽 Ingress 的變更並即時更新 

### 運作流程（以 NGINX Ingress Controller 為例）

#### 1. 啟動時註冊自己是 Ingress Controller
例如你用 Helm 或 YAML 部署一組 `nginx-ingress-controller` 的 Deployment + Service。

這組 Pod 會自動跟 K8S API Server 登記自己是 Ingress Controller。

#### 2. 持續監聽 Ingress 資源的變動
Ingress Controller 會不停地 Watch（監聽）K8S 裡所有 Namespace 下的 Ingress 物件（也會監聽相關 Service/Secret）。

每當有人新增、修改、刪除 Ingress，Controller 都會收到通知。

#### 3. 解析 Ingress 規則
Controller 讀取 Ingress 資源的內容，像是：
- 要處理哪些 host（域名）
- 哪些路徑要導到哪個 Service/Port
- 有沒有要加 SSL/TLS（憑證在哪裡）

#### 4. 自動產生反向代理設定檔
以 NGINX Controller 為例，它會把 K8S Ingress 內容「轉譯」成自己的 NGINX 設定檔（nginx.conf）。

如果規則有改，Controller 會動態 reload 設定，不需要手動重啟 Pod。

#### 5. 實際接收外部請求並代理進來
外部 HTTP/HTTPS 流量經過負載平衡器或 NodePort，被導到 Ingress Controller 的 Service。

NGINX 依照剛剛轉譯的規則，把請求導到對應的 K8S Service（再由 Service 送到 Pod）。


### 使用 Ingress 的主要優勢

- **單一進入點**：透過一個介面整合外部存取
- **成本效益**：減少對多個 LoadBalancer 服務的需求
- **進階路由**：支援基於主機和路徑的路由
- **SSL 終止**：可以集中處理 HTTPS 憑證
- **負載分散**：支援各種負載平衡演算法
