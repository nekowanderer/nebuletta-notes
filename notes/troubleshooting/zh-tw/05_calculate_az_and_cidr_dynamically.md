# 動態 AZ 和 CIDR 計算說明

[English](../en/05_calculate_az_and_cidr_dynamically.md) | [繁體中文](../zh-tw/05_calculate_az_and_cidr_dynamically.md) | [日本語](../ja/05_calculate_az_and_cidr_dynamically.md) | [回到索引](../README.md)

本文件說明 Nebuletta 專案中如何實現動態 Availability Zone (AZ) 選擇和 CIDR 區塊自動計算。

---

## 背景
- 實驗日期：2025/07/21
- 難度：🤬
- 描述：最初的實作幾乎都依靠人工計算與配置，不是很理想的解決方案。

---

### 原本的 Hard Code 方式
```hcl
# 舊的實作方式
azs = ["ap-northeast-1a", "ap-northeast-1c"]
public_subnet_cidrs = ["10.0.1.0/24", "10.0.2.0/24"]
private_subnet_cidrs = ["10.0.11.0/24", "10.0.12.0/24"]
```

**問題：**
- 無法跨 Region 使用（每個 Region 的 AZ 不同）
- 需要手動維護 AZ 清單
- 需要手動計算 CIDR 區塊
- 容易出現配置錯誤

---

## 解決方案：動態計算

### 1. 自動 AZ 偵測

```hcl
# data.tf
data "aws_availability_zones" "available" {
  state = "available"
  filter {
    name   = "opt-in-status"
    values = ["opt-in-not-required"]
  }
}

# locals.tf
available_azs = slice(
  data.aws_availability_zones.available.names, 
  0, 
  min(var.max_azs, length(data.aws_availability_zones.available.names))
)
```

**關鍵概念：**
- `state = "available"` - 只選擇可用的 AZ
- `opt-in-status = "opt-in-not-required"` - 只選擇不需額外申請的 AZ
- `min()` 函數確保不會超過實際可用的 AZ 數量

### 2. 動態 CIDR 計算

#### 步驟 1：VPC CIDR 拆解
```hcl
vpc_cidr_prefix = split("/", var.vpc_cidr)[0]  # "10.0.0.0"
vpc_cidr_mask   = tonumber(split("/", var.vpc_cidr)[1])  # 16
```

#### 步驟 2：IP 地址分段處理
```hcl
# 將 "10.0.0.0" 拆分為 ["10", "0", "0", "0"]
split(".", local.vpc_cidr_prefix)

# 取前兩段 ["10", "0"]
slice(split(".", local.vpc_cidr_prefix), 0, 2)

# 重新組合為 "10.0"
join(".", slice(split(".", local.vpc_cidr_prefix), 0, 2))
```

#### 步驟 3：Subnet CIDR 生成
```hcl
# Public subnets: 10.x.1.0/24, 10.x.2.0/24, ...
public_subnet_cidrs = [
  for i in range(length(local.available_azs)) :
  "${join(".", slice(split(".", local.vpc_cidr_prefix), 0, 2))}.${i + 1}.0/24"
]

# Private subnets: 10.x.11.0/24, 10.x.12.0/24, ...
private_subnet_cidrs = [
  for i in range(length(local.available_azs)) :
  "${join(".", slice(split(".", local.vpc_cidr_prefix), 0, 2))}.${i + 11}.0/24"
]
```

---

## 函數詳解

### Terraform 函數說明

| 函數 | 用途 | 範例 |
|------|------|------|
| `slice(list, start, end)` | 從清單中取出一段 | `slice(["a","b","c"], 0, 2)` → `["a","b"]` |
| `split(separator, string)` | 字串分割 | `split("/", "10.0.0.0/16")` → `["10.0.0.0", "16"]` |
| `join(separator, list)` | 清單合併 | `join(".", ["10","0"])` → `"10.0"` |
| `tonumber(string)` | 字串轉數字 | `tonumber("16")` → `16` |
| `min(a, b)` | 取較小值 | `min(4, 3)` → `3` |
| `length(list)` | 取得長度 | `length(["a","b"])` → `2` |
| `range(n)` | 產生數字序列 | `range(3)` → `[0, 1, 2]` |

### 完整計算範例

假設：
- `var.vpc_cidr = "10.0.0.0/16"`
- `var.max_azs = 3`
- ap-northeast-1 可用 AZ：`["ap-northeast-1a", "ap-northeast-1c", "ap-northeast-1d"]`

**計算過程：**

1. **AZ 選擇：**
   ```hcl
   available_azs = ["ap-northeast-1a", "ap-northeast-1c", "ap-northeast-1d"]
   ```

2. **CIDR 拆解：**
   ```hcl
   vpc_cidr_prefix = "10.0.0.0"
   join(".", slice(split(".", "10.0.0.0"), 0, 2)) = "10.0"
   ```

3. **Public Subnet 計算：**
   ```hcl
   i=0: "10.0.1.0/24"   # ap-northeast-1a
   i=1: "10.0.2.0/24"   # ap-northeast-1c  
   i=2: "10.0.3.0/24"   # ap-northeast-1d
   ```

4. **Private Subnet 計算：**
   ```hcl
   i=0: "10.0.11.0/24"  # ap-northeast-1a
   i=1: "10.0.12.0/24"  # ap-northeast-1c
   i=2: "10.0.13.0/24"  # ap-northeast-1d
   ```

---

## 優點總結

### ✅ 解決的問題
1. **跨 Region 相容性** - 自動適應任何 AWS Region 的 AZ 配置
2. **降低配置複雜度** - 只需指定 `max_azs` 參數
3. **減少人為錯誤** - 自動計算 CIDR，避免衝突
4. **彈性控制** - 可根據需求調整 AZ 數量
5. **容錯機制** - 自動處理 AZ 數量不足的情況

### ✅ 實際效益
- **維護性提升** - 減少 hard code 配置
- **部署靈活性** - 同一套原始碼可部署到不同 Region
- **擴展性** - 輕鬆調整 AZ 數量
- **一致性** - CIDR 分配規則統一

---

## 特殊情況說明

### ap-northeast-1 的實際狀況
- 理論上有 4 個 AZ (a, b, c, d)
- 實際只有 3 個可用 (ap-northeast-1b 已停用)
- 系統會自動偵測並使用：`["ap-northeast-1a", "ap-northeast-1c", "ap-northeast-1d"]`

### CIDR 分配規則
- **VPC CIDR**: `10.0.0.0/16` (65536 個 IP)
- **Public Subnet**: `10.0.1.0/24`, `10.0.2.0/24`, ... (每個 256 個 IP)
- **Private Subnet**: `10.0.11.0/24`, `10.0.12.0/24`, ... (每個 256 個 IP)
- **預留空間**: `10.0.0.0/24`, `10.0.4-10.0/24`, `10.0.14+.0/24` 可用於其他用途

這樣的設計確保了網路架構的靈活性和可維護性。
