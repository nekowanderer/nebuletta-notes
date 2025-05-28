# Network ACL (Stateless) vs Security Group (Stateful) 深度解析

[English](../en/03_network_acl_stateless_vs_security_group_stateful.md) | [繁體中文](03_network_acl_stateless_vs_security_group_stateful.md) | [日本語](../ja/03_network_acl_stateless_vs_security_group_stateful.md) | [返回索引](../README.md)

---

## 核心概念差異

### Stateless（無狀態）- Network ACL
**定義**：不會記住或追蹤連線的狀態和歷史

**關鍵特性**：
- 每個封包都被**獨立檢查**
- 不記住之前是否有相關的 outbound 請求
- Inbound 和 Outbound 規則**完全分開處理**
- 必須明確設定雙向規則

### Stateful（有狀態）- Security Group
**定義**：會記住並追蹤連線的狀態

**關鍵特性**：
- 記住 outbound 請求的連線狀態
- **自動允許**對應的 inbound 回應流量
- 只需要設定單向規則
- 智能處理相關聯的流量

## 實際運作對比

### 場景：EC2 訪問 `https://google.com`

#### 🔄 完整的 TCP 連線流程
```
步驟 1: [EC2:32451] ---------> [Google:443] (SYN: 我想建立連線)
步驟 2: [EC2:32451] <--------- [Google:443] (SYN-ACK: 好的，我收到了)  
步驟 3: [EC2:32451] ---------> [Google:443] (ACK: 收到，連線建立)
步驟 4: [EC2:32451] ---------> [Google:443] (HTTP Request: 給我網頁)
步驟 5: [EC2:32451] <--------- [Google:443] (HTTP Response: 這是網頁內容)
```

**說明**：
- `32451` 是 EC2 隨機選擇的 ephemeral port (1024-65535)
- `443` 是 Google 的 HTTPS 服務端口

### 🚫 Network ACL (Stateless) 的處理方式

**Outbound 規則範例**：
```hcl
egress {
  protocol   = "6"     # TCP
  rule_no    = 100
  action     = "allow"
  cidr_block = "0.0.0.0/0"
  from_port  = 443
  to_port    = 443
}
```

**處理結果**：
- ✅ 步驟 1, 3, 4 (outbound) → **允許通過**
- ❌ 步驟 2, 5 (inbound) → **被拒絕！** 

**問題根因**：
Network ACL 看到步驟 2 的封包時會想：
> "有一個從 142.251.42.142:443 到 10.0.1.10:32451 的封包，我需要檢查 inbound 規則...
> 咦，我沒有允許這個來源的規則，拒絕！"

**必須新增的 Inbound 規則**：
```hcl
# 允許 ephemeral port 回應
ingress {
  protocol   = "6"     # TCP  
  rule_no    = 90
  action     = "allow"
  cidr_block = "0.0.0.0/0"
  from_port  = 1024    # 客戶端隨機選擇的端口範圍
  to_port    = 65535
}

# 允許 HTTPS 服務端口的回應
ingress {
  protocol   = "6"     # TCP
  rule_no    = 100
  action     = "allow"
  cidr_block = "0.0.0.0/0"
  from_port  = 443
  to_port    = 443
}
```

### ✅ Security Group (Stateful) 的處理方式

**Outbound 規則範例**：
```hcl
egress {
  protocol    = "tcp"
  from_port   = 443
  to_port     = 443
  cidr_blocks = ["0.0.0.0/0"]
}
```

**處理結果**：
- ✅ 步驟 1, 3, 4 (outbound) → **允許通過**
- ✅ 步驟 2, 5 (inbound) → **自動允許！**

**智能處理過程**：
Security Group 看到步驟 2 的封包時會想：
> "有一個從 142.251.42.142:443 到 10.0.1.10:32451 的封包...
> 讓我檢查一下，噢！這是回應之前從 32451 port 發出去的連線，
> 這是合法的回應流量，允許通過！"

## 關鍵差異總結表

| 特性 | Network ACL (Stateless) | Security Group (Stateful) |
|------|-------------------------|---------------------------|
| **作用層級** | Subnet 層級 | Instance 層級 |
| **連線記憶** | ❌ 不記住連線狀態 | ✅ 記住連線狀態 |
| **規則需求** | 必須設定雙向規則 | 只需設定單向規則 |
| **回應處理** | 需要明確允許回應流量 | 自動允許相關回應 |
| **預設行為** | 預設 deny all | 預設 deny inbound, allow outbound |
| **規則評估** | 按照 rule number 順序 | 評估所有規則（OR 邏輯）|
| **適用範圍** | 整個 subnet 內的所有資源 | 特定的 EC2 instance |

## Ephemeral Ports 的重要性

### 什麼是 Ephemeral Ports？

**定義**：暫時性端口，客戶端在發起連線時隨機選擇的端口號

**範圍**：
- Linux/Windows: 1024-65535
- 部分較新系統: 32768-65535

### 為什麼需要允許 Ephemeral Ports？

```
實際連線範例：
EC2 (IP: 10.0.1.10, Port: 32451) → Google (IP: 142.251.42.142, Port: 443)
EC2 (IP: 10.0.1.10, Port: 32451) ← Google (IP: 142.251.42.142, Port: 443)
```

**Network ACL 看到的回應封包**：
- **來源**：142.251.42.142:443
- **目的地**：10.0.1.10:32451

如果沒有允許 port 32451 的 inbound 規則，這個回應就會被擋掉。

### 常見的 Ephemeral Port 設定

```hcl
# 允許所有 ephemeral ports (建議用於大部分情況)
ingress {
  protocol   = "6"
  rule_no    = 90
  action     = "allow"
  cidr_block = "0.0.0.0/0"
  from_port  = 1024
  to_port    = 65535
}

# 只允許常見範圍 (較嚴格)
ingress {
  protocol   = "6"
  rule_no    = 90
  action     = "allow"
  cidr_block = "0.0.0.0/0"
  from_port  = 32768
  to_port    = 65535
}
```

## 常見問題與解決方案

### 問題 1：能 ping 出去但沒有回應

**症狀**：
```bash
$ ping www.google.com
# 沒有任何回應，看起來像是網路不通
```

**原因**：
```
ICMP Echo Request  (outbound) ✅ → 被允許
ICMP Echo Reply    (inbound)  ❌ → 被 Network ACL 阻擋
```

**解決方案**：
```hcl
# 允許 ICMP 回應
ingress {
  protocol   = "1"  # ICMP
  rule_no    = 80
  action     = "allow"
  cidr_block = "0.0.0.0/0"
  from_port  = 0
  to_port    = 0
  icmp_type  = -1
  icmp_code  = -1
}
```

### 問題 2：HTTPS 連線卡住

**症狀**：
```bash
$ curl https://google.com
# 卡住不動，最後 timeout
```

**原因**：
- TCP SYN 可以送出
- TCP SYN-ACK 回應被阻擋
- 無法完成三次握手

**解決方案**：
```hcl
# 允許 ephemeral port 回應
ingress {
  protocol   = "6"
  rule_no    = 90
  action     = "allow"
  cidr_block = "0.0.0.0/0"
  from_port  = 1024
  to_port    = 65535
}
```

### 問題 3：內部服務無法訪問

**症狀**：
同一個 VPC 內的服務互相無法連線

**解決方案**：
```hcl
# 允許 VPC 內部通信
ingress {
  protocol   = "-1"  # 所有協定
  rule_no    = 110
  action     = "allow"
  cidr_block = var.vpc_cidr  # 例如：10.0.0.0/16
  from_port  = 0
  to_port    = 0
}

egress {
  protocol   = "-1"
  rule_no    = 120
  action     = "allow"
  cidr_block = var.vpc_cidr
  from_port  = 0
  to_port    = 0
}
```

## 最佳實踐建議

### Network ACL 設計原則

1. **優先考慮雙向通信**
   - 每個 outbound 規則都要有對應的 inbound 規則
   - 特別注意 ephemeral port 範圍

2. **使用較低的 rule number 給重要規則**
   - Network ACL 按照 rule number 順序評估
   - 較低的 number 會先被評估

3. **明確拒絕不需要的流量**
   - 使用明確的 deny 規則
   - 避免依賴預設的 deny all

### Security Group 設計原則

1. **最小權限原則**
   - 只開放必要的端口和來源
   - 定期審查和清理不需要的規則

2. **使用描述性的規則名稱**
   - 清楚標註每個規則的用途
   - 方便後續維護和除錯

## 記憶口訣

### Network ACL (Stateless)
> **「健忘症患者」**
> 
> "我不記得你剛才說了什麼，每個封包都要重新檢查所有規則！
>  你要進來？給我看通行證！你要出去？也給我看通行證！"

### Security Group (Stateful)
> **「貼心管家」**
> 
> "我記得你剛才的請求，既然你已經通過驗證出去了，
>  你的回應當然可以直接進來，不用再檢查一次！"

## 總結

理解 **Stateless** vs **Stateful** 的差異是掌握 AWS 網路安全的關鍵：

- **Network ACL** 像是嚴格的邊境檢查站，每個人進出都要檢查證件
- **Security Group** 像是智能的門禁系統，記住誰出去了，自動讓他們回來

當你遇到「能送出但收不到回應」的問題時，十之八九是 Network ACL 的 stateless 特性造成的！ 