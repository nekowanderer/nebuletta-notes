# AWS 負載均衡指南

[English](../en/19_aws_load_balancing.md) | [繁體中文](../zh-tw/19_aws_load_balancing.md) | [日本語](../ja/19_aws_load_balancing.md) | [回到索引](../README.md)

### AWS 負載均衡器類型比較

AWS 提供三種主要的負載均衡器類型，每種都有其特定的使用場景和功能特性：

| 類型 | 適用場景 | 支援協定 | 運作層級 | 主要特色 |
|------|----------|----------|----------|----------|
| **Application Load Balancer (ALB)** | HTTP/HTTPS 應用程式 | HTTP、HTTPS、WebSocket | Layer 7 (應用層) | 路徑路由、主機路由、Lambda 整合 |
| **Network Load Balancer (NLB)** | 高效能 TCP/UDP 流量 | TCP、UDP、TLS | Layer 4 (傳輸層) | 超低延遲、靜態 IP、極高輸送量 |
| **Gateway Load Balancer (GLB)** | 第三方網路設備 | GENEVE | Layer 3 (網路層) | 防火牆、IDS/IPS、DPI 設備整合 |

### 負載均衡器核心功能

#### 流量分配演算法
- **Round Robin**：依序分配請求到各個目標
- **Least Outstanding Requests**：分配到處理中請求數最少的目標
- **Flow Hash**：基於來源 IP、目標 IP、協定的雜湊值分配

#### 健康檢查機制
- **健康檢查路徑**：定期檢查目標是否正常回應
- **健康門檻**：連續成功次數達到門檻才標記為健康
- **不健康門檻**：連續失敗次數達到門檻才標記為不健康
- **檢查間隔**：健康檢查的頻率設定

---

## Target Group 目標群組

### Target Group 與 ALB 的關聯

#### 基本關係
- **ALB 不能直接路由到 EC2 或服務**
- **必須透過 Target Group 作為中介**
- Target Group 是 ALB 的「目標容器」

#### 關聯架構圖
```
Internet → ALB → Listener → Rules → Target Group → Targets (EC2/IP/Lambda)
```

#### ALB Listener 與 Target Group 整合
```bash
# ALB 的 Listener 會將流量轉發到 Target Group
$ aws elbv2 create-listener \
    --load-balancer-arn arn:aws:elasticloadbalancing:region:account:loadbalancer/app/my-alb/1234567890123456 \
    --protocol HTTP \
    --port 80 \
    --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:region:account:targetgroup/my-targets/1234567890123456
```

#### 一個 ALB 多個 Target Group 的應用
- **路徑路由**：`/api/*` → API Target Group
- **主機路由**：`admin.example.com` → Admin Target Group
- **預設路由**：其他所有請求 → Default Target Group

### 實際應用範例

#### 微服務架構
```
ALB
├── /api/users/* → User Service Target Group
├── /api/orders/* → Order Service Target Group  
├── /admin/* → Admin Panel Target Group
└── /* → Frontend Target Group
```

#### 藍綠部署
```
ALB
├── 90% 流量 → Production Target Group (藍)
└── 10% 流量 → Staging Target Group (綠)
```

### 負載均衡流程

1. **ALB 接收請求**
2. **Listener 檢查埠號和協定**
3. **Rules 決定轉發到哪個 Target Group**
4. **Target Group 根據演算法選擇健康的 Target**
5. **將請求轉發到選定的 Target**

### Target Group 在 ALB 中的角色

| 功能 | Target Group 的作用 |
|------|-------------------|
| **健康檢查** | 持續監控後端服務狀態 |
| **負載分配** | 根據演算法分配流量 |
| **服務發現** | 動態註冊/取消註冊目標 |
| **會話黏性** | 維持用戶會話一致性 |

### 目標群組類型

#### Instance 目標群組
- **目標類型**：EC2 實例
- **註冊方式**：直接指定 EC2 instance ID
- **適用場景**：傳統 EC2 架構，需要與 Auto Scaling Group 整合

#### IP 目標群組
- **目標類型**：IP 地址
- **註冊方式**：直接指定 IP 地址和埠號
- **適用場景**：容器化應用、微服務架構、跨 VPC 通訊

#### Lambda 目標群組
- **目標類型**：Lambda 函數
- **註冊方式**：指定 Lambda 函數 ARN
- **適用場景**：Serverless 架構，事件驅動的 HTTP API

### 目標群組健康檢查設定

```bash
# 健康檢查設定範例
$ aws elbv2 modify-target-group \
    --target-group-arn arn:aws:elasticloadbalancing:region:account:targetgroup/my-targets/1234567890123456 \
    --health-check-path /health \
    --health-check-interval-seconds 30 \
    --health-check-timeout-seconds 5 \
    --healthy-threshold-count 2 \
    --unhealthy-threshold-count 5
```

### 目標群組屬性配置

| 屬性 | 說明 | 預設值 | 建議設定 |
|------|------|--------|----------|
| **deregistration_delay.timeout_seconds** | 取消註冊等待時間 | 300 秒 | 根據應用程式關閉時間調整 |
| **stickiness.enabled** | 啟用會話黏性 | false | 有狀態應用程式設為 true |
| **stickiness.type** | 黏性類型 | lb_cookie | 應用程式 cookie 或負載均衡器 cookie |
| **load_balancing.algorithm.type** | 負載均衡演算法 | round_robin | 根據應用程式特性選擇 |

---

## Trust Stores 信任存放區

### 信任存放區概念

信任存放區是 AWS Application Load Balancer 的一個進階功能，用於管理和驗證用戶端憑證：

#### 主要功能
- **用戶端憑證驗證**：驗證連接到 ALB 的用戶端憑證
- **mTLS 支援**：實現雙向 TLS 認證
- **憑證鏈驗證**：驗證完整的憑證鏈
- **憑證撤銷檢查**：支援 CRL 和 OCSP 驗證

#### 使用場景
- **企業內部 API**：需要強制用戶端憑證認證
- **B2B 整合**：與合作夥伴的安全 API 通訊
- **合規要求**：滿足特定行業的安全標準
- **零信任架構**：實現端到端的憑證驗證

### 信任存放區設定流程

#### 1. 建立信任存放區
```bash
$ aws elbv2 create-trust-store \
    --name my-trust-store \
    --ca-certificates-bundle-s3-bucket my-ca-bucket \
    --ca-certificates-bundle-s3-key ca-bundle.pem
```

#### 2. 關聯到監聽器
```bash
$ aws elbv2 modify-listener \
    --listener-arn arn:aws:elasticloadbalancing:region:account:listener/app/my-load-balancer/1234567890123456/1234567890123456 \
    --mutual-authentication Mode=verify,TrustStoreArn=arn:aws:elasticloadbalancing:region:account:truststore/my-trust-store/1234567890123456
```

#### 3. 憑證格式要求
- **格式**：PEM 格式的 CA 憑證
- **大小限制**：最大 1MB
- **憑證數量**：每個信任存放區最多 10,000 個憑證
- **更新方式**：透過 S3 上傳新的憑證包

### 信任存放區最佳實務

#### 安全性考量
1. **定期更新憑證**：建立憑證輪換機制
2. **最小權限原則**：只授予必要的憑證存取權限
3. **監控和記錄**：記錄憑證驗證事件
4. **備份策略**：定期備份憑證和信任存放區設定

#### 營運管理
1. **憑證生命週期管理**：追蹤憑證到期時間
2. **自動化部署**：使用 Infrastructure as Code 管理
3. **測試環境**：建立測試用的信任存放區
4. **錯誤處理**：設定憑證驗證失敗的處理邏輯

---

## 負載均衡器進階功能

### 路由規則設定

#### ALB 路由規則類型
- **路徑路由**：根據 URL 路徑分配流量
- **主機路由**：根據 Host header 分配流量
- **標頭路由**：根據 HTTP 標頭分配流量
- **查詢字串路由**：根據查詢參數分配流量

#### 路由規則範例
```bash
# 建立路徑路由規則
$ aws elbv2 create-rule \
    --listener-arn arn:aws:elasticloadbalancing:region:account:listener/app/my-load-balancer/1234567890123456/1234567890123456 \
    --conditions Field=path-pattern,Values="/api/*" \
    --priority 100 \
    --actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:region:account:targetgroup/api-targets/1234567890123456
```

### 跨區域負載均衡

#### 跨區域負載均衡設定
- **預設行為**：ALB 和 GLB 預設啟用，NLB 預設停用
- **效能影響**：可能增加跨區域流量成本
- **高可用性**：提升故障容錯能力

#### 成本優化考量
- **區域感知路由**：優先路由到相同區域的目標
- **流量分析**：監控跨區域流量模式
- **成本監控**：設定跨區域流量警報

### 整合 AWS 服務

#### Auto Scaling 整合
- **動態擴展**：根據負載自動調整目標數量
- **健康檢查整合**：不健康的實例自動替換
- **暖機時間**：新實例準備時間設定

#### CloudWatch 監控
- **關鍵指標**：請求計數、延遲、錯誤率
- **自訂警報**：設定效能和可用性警報
- **日誌分析**：存取日誌到 S3 進行分析

#### WAF 整合
- **安全防護**：Web 應用程式防火牆規則
- **DDoS 保護**：與 AWS Shield 整合
- **地理封鎖**：根據地理位置控制存取

---

## 負載均衡器最佳實務

### 效能優化
1. **連接重用**：啟用 Keep-Alive 連接
2. **壓縮設定**：啟用 HTTP 壓縮
3. **快取策略**：設定適當的快取標頭
4. **SSL 終止**：在負載均衡器層級處理 SSL

### 安全性強化
1. **SSL/TLS 政策**：使用最新的安全政策
2. **存取控制**：設定安全群組規則
3. **憑證管理**：使用 AWS Certificate Manager
4. **監控日誌**：啟用存取日誌記錄

### 故障排除
1. **健康檢查失敗**：檢查目標健康狀態
2. **連接超時**：調整超時設定
3. **502/503 錯誤**：檢查後端服務狀態
4. **SSL 握手失敗**：驗證憑證設定

### 成本最佳化
1. **負載均衡器類型選擇**：根據需求選擇適當類型
2. **目標群組設定**：最佳化目標數量和分配
3. **監控使用量**：定期檢查流量模式
4. **預留容量**：考慮使用 Savings Plans
