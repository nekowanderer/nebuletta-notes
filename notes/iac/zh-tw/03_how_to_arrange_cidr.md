# CIDR 規劃與計算方式

[English](../en/03_how_to_arrange_cidr.md) | [繁體中文](../zh-tw/03_how_to_arrange_cidr.md) | [日本語](../ja/03_how_to_arrange_cidr.md) | [回到索引](../README.md)

## CIDR 基本概念

CIDR (Classless Inter-Domain Routing) 是一種 IP 地址分配和路由的方法，用於更有效地管理網路資源。

### 表示法解析

```
10.0.0.0/16
    ↑     ↑
   IP   前綴長度
```

**前綴長度**表示網路部分的位元數：
- `/16` = 前16個位元是網路部分
- `/24` = 前24個位元是網路部分
- `/32` = 主機地址（單一 IP）

---

## 位元組織與計算

### 1. IP 地址結構
```
IP 地址：32 位元 = 4 個位元組

範例：10.0.0.0
二進位：00001010.00000000.00000000.00000000
          ↑        ↑        ↑        ↑
        第1位元組  第2位元組  第3位元組  第4位元組
```

### 2. 網路與主機部分劃分
```
10.0.0.0/16 的劃分：

網路部分：前16位元 (00001010.00000000)
主機部分：後16位元 (.00000000.00000000)

可變動範圍：10.0.0.0 ~ 10.0.255.255
```

---

## 可用 IP 數量計算

### 計算公式
**可用 IP 數量 = 2^(32-前綴長度) - 2**

> **為什麼要減 2？**
> - 第一個 IP：網路地址（不可用）
> - 最後一個 IP：廣播地址（不可用）

### 常見 CIDR 容量表

| CIDR | 網路位元 | 主機位元 | 總 IP 數 | 可用 IP 數 | 用途 |
|------|----------|----------|----------|------------|------|
| /16  | 16       | 16       | 65,536   | 65,534     | 大型網路 |
| /24  | 24       | 8        | 256      | 254        | 一般子網路 |
| /25  | 25       | 7        | 128      | 126        | 小型子網路 |
| /26  | 26       | 6        | 64       | 62         | 微型子網路 |
| /27  | 27       | 5        | 32       | 30         | 極小子網路 |
| /28  | 28       | 4        | 16       | 14         | 設備群組 |

### 實際計算範例

#### 範例 1：VPC 規劃
```
VPC: 10.0.0.0/16

計算過程：
主機位元：32 - 16 = 16 位元
可用 IP：2^16 - 2 = 65,534 個
IP 範圍：10.0.0.1 ~ 10.0.255.254
```

#### 範例 2：子網路規劃
```
子網路: 10.0.1.0/24

計算過程：
主機位元：32 - 24 = 8 位元
可用 IP：2^8 - 2 = 254 個
IP 範圍：10.0.1.1 ~ 10.0.1.254
```

---

## 子網路劃分計算

### 從大網路劃分小網路

#### 範例：從 /16 劃分出 /24
```
原始網路：10.0.0.0/16

可劃分的 /24 子網路數量：
2^(24-16) = 2^8 = 256 個

子網路列表：
10.0.0.0/24    → 10.0.0.1   ~ 10.0.0.254   (254個IP)
10.0.1.0/24    → 10.0.1.1   ~ 10.0.1.254   (254個IP)
10.0.2.0/24    → 10.0.2.1   ~ 10.0.2.254   (254個IP)
...
10.0.255.0/24  → 10.0.255.1 ~ 10.0.255.254 (254個IP)
```

### 子網路邊界檢查

檢查一個 IP 是否為合法的子網路起始地址：

```bash
# 檢查 10.0.11.0/24 是否合法

Step 1: 分析 IP 地址結構
10.0.11.0/24
- 網路部分：前24位元 (10.0.11)
- 主機部分：後8位元 (0)

Step 2: 檢查主機部分是否為0
第4個位元組 = 0 = 00000000
主機部分全為0

# 結論：10.0.11.0/24 是合法的子網路起始地址
```

#### 另一個範例：檢查不合法的情況
```bash
# 檢查 10.0.11.5/24 是否合法

Step 1: 分析 IP 地址結構  
10.0.11.5/24
- 網路部分：前24位元 (10.0.11)
- 主機部分：後8位元 (5)

Step 2: 檢查主機部分是否為0
第4個位元組 = 5 = 00000101
主機部分不為0

# 結論：10.0.11.5/24 不是合法的子網路起始地址
# 正確的子網路應該是：10.0.11.0/24
```

#### 更複雜的範例：檢查 /26 子網路
```bash
# 檢查 10.0.1.64/26 是否合法

Step 1: 分析 /26 的結構
/26 = 26位元網路部分 + 6位元主機部分
主機部分：後6位元必須為0

Step 2: 檢查第4個位元組
64 = 01000000 (二進位)
後6位元 = 000000 ✓ (全為0)

# 結論：10.0.1.64/26 是合法的子網路

# /26 的合法子網路邊界：
# 10.0.1.0/26   (0   = 00000000)
# 10.0.1.64/26  (64  = 01000000) 
# 10.0.1.128/26 (128 = 10000000)
# 10.0.1.192/26 (192 = 11000000)
```

---

## 多環境 CIDR 規劃實務

### 環境隔離策略

推薦的多環境規劃：

```
Dev：    10.0.x.x/16  (10.0.0.0   ~ 10.0.255.255)
Staging： 10.1.x.x/16  (10.1.0.0   ~ 10.1.255.255)
Production：  10.2.x.x/16  (10.2.0.0   ~ 10.2.255.255)
Testing：     10.3.x.x/16  (10.3.0.0   ~ 10.3.255.255)
```

### 具體規劃範例

#### Dev 環境：10.0.0.0/16
```
VPC 總容量：65,534 個 IP

Public 子網路：
├── AZ-a: 10.0.1.0/24  → 254 個 IP
└── AZ-b: 10.0.2.0/24  → 254 個 IP

Private 子網路：
├── AZ-a: 10.0.11.0/24 → 254 個 IP  
└── AZ-b: 10.0.12.0/24 → 254 個 IP

Database 子網路：
├── AZ-a: 10.0.21.0/24 → 254 個 IP
└── AZ-b: 10.0.22.0/24 → 254 個 IP

總使用：6 × 254 = 1,524 個 IP
使用率：1,524 / 65,534 = 2.3%
```

#### 完整環境規劃
```
Dev 環境 (10.0.x.x/16)
├── Public:   10.0.1.0/24,  10.0.2.0/24
├── Private:  10.0.11.0/24, 10.0.12.0/24
└── Database: 10.0.21.0/24, 10.0.22.0/24

Staging 環境 (10.1.x.x/16)
├── Public:   10.1.1.0/24,  10.1.2.0/24
├── Private:  10.1.11.0/24, 10.1.12.0/24
└── Database: 10.1.21.0/24, 10.1.22.0/24

Production 環境 (10.2.x.x/16)
├── Public:   10.2.1.0/24,  10.2.2.0/24
├── Private:  10.2.11.0/24, 10.2.12.0/24
└── Database: 10.2.21.0/24, 10.2.22.0/24
```

---

## 規劃優勢與注意事項

### 這種規劃的優勢

1. **避免 IP 衝突**
   - 不同環境使用不同的 /16 區塊
   - VPC Peering 或 Transit Gateway 連接時不會衝突

2. **保持一致的模式**
   - 所有環境採用相同的子網路編號規則
   - 易於記憶和維護

3. **容易識別環境**
   ```
   看到 10.0.x.x → 知道是 Dev 環境
   看到 10.1.x.x → 知道是 Staging 環境
   看到 10.2.x.x → 知道是 Production 環境
   ```

4. **充足的擴展空間**
   - 每個環境有 65,534 個可用 IP
   - 可以新增更多子網路而不重新規劃

### 注意事項

1. **預留足夠空間**
   ```bash
   # 避免過度劃分
   ❌ 不好：用完所有 /24 子網路
   ✅ 良好：只使用需要的子網路，保留空間給未來
   ```

2. **考慮路由需求**
   - 不同子網路間的通訊需求
   - 路由表的複雜度管理

3. **安全群組規劃**
   - 基於 CIDR 範圍設定安全規則
   - 最小權限原則

---

## 實用工具與技巧

### 快速計算技巧

```bash
# 記住常用的容量
/16 = 65K 個 IP   (適合 VPC)
/24 = 256 個 IP   (適合一般子網路)
/26 = 64 個 IP    (適合小型微服務)
/28 = 16 個 IP    (適合資料庫叢集)

# CIDR 計算器指令
$ ipcalc 10.0.0.0/16
$ sipcalc 10.0.1.0/24
```

### 檢查工具
```python
# Python 範例
import ipaddress

network = ipaddress.IPv4Network('10.0.0.0/16')
print(f"網路地址: {network.network_address}")
print(f"廣播地址: {network.broadcast_address}")
print(f"可用 IP 數: {network.num_addresses - 2}")

# 檢查子網路
subnet = ipaddress.IPv4Network('10.0.1.0/24')
print(f"是否包含在內: {subnet.subnet_of(network)}")
```

---

## 總結

良好的 CIDR 規劃能夠：

1. **提高維運效率**：清楚的命名規則與結構
2. **避免未來衝突**：充足的地址空間預留
3. **簡化網路管理**：一致的架構模式
4. **支援環境擴展**：彈性的成長空間

記住：**規劃時要考慮現在的需求，但也要為未來留下空間！**
