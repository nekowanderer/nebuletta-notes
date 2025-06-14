# Network ACL 對 Private Subnet 通信影響的除錯經驗

[English](../en/01_network_acl_private_subnet_troubleshooting.md) | [繁體中文](01_network_acl_private_subnet_troubleshooting.md) | [日本語](../ja/01_network_acl_private_subnet_troubleshooting.md) | [返回索引](../README.md)

---

## 背景
- 實驗日期: 2025/05/21
- 難度：🤬🤬🤬
- 描述: 在 AWS VPC 環境中，private subnet 內的 EC2 實例無法正常與外部網路通信，即使所有看似相關的設定都正確。

## 遇到的現象

- EC2 透過 Session Manager 可以正常連線
- `ping www.google.com` 沒有任何回應
- `curl https://google.com` 會卡住無法完成連線

## 除錯過程

### 第一階段：確認基礎設施設定

已確認以下各項目都設定正確：

| 項目 | 狀態 | 說明 |
|------|------|------|
| EC2 Instance | ✅ | 沒有 Public IP，正確位於 Private Subnet |
| Private Subnet | ✅ | 有對應的 Route Table |
| Route Table | ✅ | 有設定 `0.0.0.0/0` ➝ NAT Gateway |
| NAT Gateway | ✅ | 已部署於 Public Subnet 並有 Elastic IP |
| Security Group | ✅ | Outbound 為 `0.0.0.0/0` all traffic |

### 第二階段：DNS 解析測試

執行 `nslookup google.com` 得到正常結果：

```bash
Server:         10.0.0.2
Address:        10.0.0.2#53

Non-authoritative answer:
Name:   google.com
Address: 142.251.42.142
Name:   google.com
Address: 2404:6800:4004:827::200e
```

**結論**：DNS 解析正常，表示 VPC 的 DNS 設定與 `/etc/resolv.conf` 都沒問題。

### 第三階段：排除可能原因

由於 DNS 正常但 HTTPS 連線失敗，可能的原因包括：

1. ❌ NAT Gateway 沒有通到 Internet Gateway
2. ❌ NAT Gateway 所在的 Public Subnet 的 Route Table 沒有設 `0.0.0.0/0` ➝ IGW
3. ✅ **NACL（Network ACL）阻擋連線** ← 真正的問題！
4. ❌ EC2 的 Source/Dest Check 沒關閉（僅適用於 NAT Instance）

## 問題根因

### Network ACL 設定錯誤

檢查綁定在 private subnet 上的 Network ACL 規則：

**Inbound rules（有問題）：**
| Rule # | Source | Protocol | Port | Allow/Deny |
|--------|--------|----------|------|------------|
| 100 | 10.0.0.0/16 | All | All | ✅ Allow |
| * | 0.0.0.0/0 | All | All | ❌ **Deny** ← 問題在這裡！|

**Outbound rules（看似正常但有陷阱）：**
| Rule # | Destination | Protocol | Port | Allow/Deny |
|--------|-------------|----------|------|------------|
| 100 | 0.0.0.0/0 | All | All | ✅ Allow |
| * | 0.0.0.0/0 | All | All | ❌ **Deny** ← 仍會生效！|

## 問題原理解析

### ICMP（Ping）的工作流程

1. **Outbound ICMP Request**：
   - EC2 發送 ping 封包到外部網址
   - 符合 Outbound Rule 100：`0.0.0.0/0 Allow`
   - 封包可以順利送出

2. **Inbound ICMP Reply**：
   - 外部伺服器回傳 ping 回應
   - 來源 IP 不是 VPC CIDR (`10.0.0.0/16`)
   - 被 Inbound Rule `* 0.0.0.0/0 Deny` 阻擋
   - **回應封包無法進入，造成 ping 失敗**

### 其他協定的影響

同樣的問題也影響：
- HTTPS 連線 (port 443)
- HTTP 連線 (port 80)  
- 任何需要外部回應的通信

## 解決方案

### 需要新增的 Inbound Rules

為了讓 private subnet 中的 EC2 能正常對外通信，必須在 Network ACL 中新增以下 inbound 規則：

1. **允許 ICMP 回應**：
   ```hcl
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
   - 這個設定在 Terraform 裡面要注意撰寫的方式，譬如，如下方式是錯誤的：
     - <image src=../images/01_wrong_config_for_icmp.jpg />
   - 應該要這樣寫：
     - <image src=../images/01_correct_config_for_icmp.jpg />
   - 長出來就是這樣：
     - <image src=../images/01_icmp_acls_rule_result.jpg />

2. **允許 Ephemeral Port 回應流量**：
   ```hcl
   ingress {
     protocol   = "6"  # TCP
     rule_no    = 90
     action     = "allow"
     cidr_block = "0.0.0.0/0"
     from_port  = 1024
     to_port    = 65535
   }
   ```

3. **允許 HTTPS 回應**：
   ```hcl
   ingress {
     protocol   = "6"  # TCP
     rule_no    = 100
     action     = "allow" 
     cidr_block = "0.0.0.0/0"
     from_port  = 443
     to_port    = 443
   }
   ```

## 關鍵學習重點

### Network ACL vs Security Group

| 特性 | Network ACL | Security Group |
|------|-------------|----------------|
| 作用層級 | Subnet 層級 | Instance 層級 |
| 狀態 | **Stateless** | Stateful |
| 回應處理 | 需要明確允許回應流量 | 自動允許回應流量 |
| 預設行為 | 預設 deny all | 預設 deny inbound, allow outbound |

### Stateless 的重要性

- **Stateless** 表示 Network ACL 不會記住連線狀態
- Outbound 允許不代表 Inbound 回應也會被允許
- **必須明確設定雙向規則**

### Ephemeral Port 的重要性

- TCP 連線建立時，客戶端會使用隨機的高端口號 (1024-65535)
- 伺服器回應會發送到這些高端口
- 如果 inbound 沒有允許這些端口，連線會失敗

## 預防措施

1. **設計 Network ACL 時優先考慮雙向通信**
2. **不要忽略 ephemeral port 範圍**
3. **測試時要同時驗證 outbound 和 inbound 流量**
4. **記住 Network ACL 是 stateless 的特性**

## 結論

這次的除錯經驗提醒我們，在 AWS 網路設定中，Network ACL 是一個容易被忽略但影響重大的元件。特別是 private subnet 的設定，必須仔細考慮所有可能的回應流量，才能確保服務能正常運作。

**記住**：能 ping 出去不代表能收到回應，stateless 的 Network ACL 需要明確的雙向規則設定！ 