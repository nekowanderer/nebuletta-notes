# Kubernetes Ingress 中的 L4/L7 網路概念

[English](../en/27_k8s_ingress_l4_l7.md) | [繁體中文](../zh-tw/27_k8s_ingress_l4_l7.md) | [日本語](../ja/27_k8s_ingress_l4_l7.md) | [回到索引](../README.md)

### OSI 網路模型基礎

在討論 Kubernetes Ingress 的 L4/L7 概念之前，需要先了解 OSI 網路模型的基本架構：

- **Layer 1 (實體層)**：電纜、光纖等實體連接
- **Layer 2 (資料連結層)**：MAC 位址、乙太網路框架
- **Layer 3 (網路層)**：IP 位址、路由
- **Layer 4 (傳輸層)**：TCP/UDP 協定、連接埠
- **Layer 5-6 (會話/呈現層)**：加密、壓縮
- **Layer 7 (應用層)**：HTTP、HTTPS、FTP 等應用協定

### Layer 4 (L4) 負載平衡

Layer 4 負載平衡運作在傳輸層，主要特徵：

**運作原理：**
- 基於 IP 位址和連接埠進行路由決策
- 不檢查封包內容，只看來源和目的地資訊
- 使用 TCP/UDP 連接資訊進行負載分散

**技術特點：**
- **高效能**：處理速度快，延遲低
- **協定無關**：可處理任何基於 TCP/UDP 的流量
- **簡單路由**：僅基於網路資訊進行轉發

**在 Kubernetes 中的應用：**
```
外部流量 → LoadBalancer Service → 基於 IP:Port → 後端 Pod
```

**使用場景：**
- 需要高吞吐量的應用
- 非 HTTP 協定的服務（如資料庫、TCP 服務）
- 簡單的負載分散需求

### Layer 7 (L7) 負載平衡

Layer 7 負載平衡運作在應用層，具有更智慧的路由能力：

**運作原理：**
- 檢查 HTTP/HTTPS 請求的完整內容
- 基於 URL 路徑、主機名、請求標頭等進行路由
- 可以修改請求和回應內容

**技術特點：**
- **智慧路由**：基於應用層資訊做複雜路由決策
- **內容感知**：可以根據請求內容進行處理
- **協定特定**：主要處理 HTTP/HTTPS 流量

**在 Kubernetes 中的應用：**
```
HTTP 請求 → Ingress Controller → 基於 Host/Path → 特定 Service → Pod
```

**路由範例：**
- `api.example.com/users` → User Service
- `api.example.com/orders` → Order Service
- `admin.example.com/*` → Admin Service

### Kubernetes 中的 組態方式

**L4 服務（LoadBalancer Service）：**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-l4
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: my-app
```

**L7 服務（Ingress）：**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-l7
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 80
```

### 實際應用場景比較

| 考量面向 | L4 負載平衡 | L7 負載平衡 |
|----------|-------------|-------------|
| **選擇時機** | • 需要最高效能和最低延遲<br>• 處理非 HTTP 協定<br>• 簡單的負載分散需求<br>• 不需要基於內容的路由 | • 需要基於 URL 路徑或主機名的路由<br>• 要實作 SSL 終止<br>• 需要應用層的健康檢查<br>• 要進行請求修改或重寫<br>• 實作微服務架構的 API Gateway |
| **實際應用場景** | • 資料庫連線池（MySQL、PostgreSQL）<br>• TCP 長連接服務<br>• 遊戲伺服器<br>• 即時通訊系統 | • Web 應用程式的路由<br>• RESTful API 的服務分流<br>• 基於使用者的路由（A/B 測試）<br>• SSL 憑證集中管理 |

### 效能與功能的權衡

| 特性 | L4 負載平衡 | L7 負載平衡 |
|------|-------------|-------------|
| 處理速度 | 快 | 較慢 |
| 功能豐富度 | 基本 | 豐富 |
| 路由智慧 | 簡單 | 複雜 |
| 協定支援 | 所有 TCP/UDP | 主要 HTTP/HTTPS |
| 資源消耗 | 低 | 高 |
| 設定複雜度 | 簡單 | 複雜 |

### 比較白話的比喻
| 角色   | 類比                                 | 檢查項目                                          | 對應 Ingress 層級 |
| ---- | ---------------------------------- | --------------------------------------------- | ------------- |
| 管制閘門 | 只認得「這張門禁卡可不可以打開這道門」──確認樓層、門號       | 卡片號碼、門號 → 就像 **IP 位址 + 埠號**                   | **L4**        |
| 櫃臺接待 | 進一步確認「你要找哪個部門、預約內容是什麼」──看訪客資料、預約目的 | 造訪部門、訪客姓名、行程備註 → 就像 **HTTP Host、Path、Header** | **L7**        |
- L4 Ingress：像閘門只看「卡號配不配這道門」──確認 TCP/UDP + 埠號就放行。
- L7 Ingress：像櫃臺除了刷卡，還要看你要去幾樓、部門名稱，甚至填訪客資料單才轉送。
