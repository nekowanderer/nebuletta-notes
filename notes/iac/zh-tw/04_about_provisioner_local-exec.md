# Terraform Provisioner 與條件邏輯說明

[English](../en/04_about_provisioner_local-exec.md) | [繁體中文](../zh-tw/04_about_provisioner_local-exec.md) | [日本語](../ja/04_about_provisioner_local-exec.md) | [回到索引](../README.md)

## 概述

本文說明 Terraform 中 `provisioner "local-exec"` 的使用方式，以及 Terraform 如何透過 `count` 和三元運算子來實現條件邏輯。

## Provisioner "local-exec" 詳解

### 基本概念

`provisioner "local-exec"` 是 Terraform 的內建 provisioner，用於在資源生命週期的特定時機執行本地 shell 命令。

### 語法結構

```hcl
provisioner "local-exec" {
  command = "要執行的命令"
  # 其他可選參數
}
```

### 執行時機

- **create**：資源建立後執行（預設）
- **destroy**：資源刪除前執行
- **update**：資源更新後執行

### 保留字說明

`local-exec` 是 Terraform 的**保留字**，不是自訂名詞。它是 Terraform 內建的 provisioner 類型之一。

## 條件邏輯實現方式

### 使用 count 進行條件建立

```hcl
resource "null_resource" "vpc_config_validation" {
  count = local.vpc_config_valid ? 0 : 1

  provisioner "local-exec" {
    command = "echo 'Error: Invalid VPC configuration. Please ensure networking module is deployed and vpc_config is properly configured.' && exit 1"
  }
}
```

### 邏輯分析

| `local.vpc_config_valid` | `count` | 結果 |
|--------------------------|---------|------|
| `true` | `0` | **不建立資源** → **不執行 provisioner** |
| `false` | `1` | **建立資源** → **執行 provisioner** |

### 三元運算子語法

```hcl
count = local.vpc_config_valid ? 0 : 1
```

等同於：
```hcl
if local.vpc_config_valid == true:
  count = 0  # 不建立資源
else:
  count = 1  # 建立 1 個資源
```

## Terraform 條件邏輯設計哲學

### 為什麼不使用傳統 if-else？

1. **宣告式語言特性**
   - Terraform 是宣告式（declarative）而非命令式（imperative）語言
   - 專注於描述「目標狀態」而非「執行過程」

2. **狀態管理優先**
   - 專注於管理基礎設施的「狀態」
   - 確保資源狀態的一致性和可預測性

3. **冪等性保證**
   - 無論執行多少次，結果都相同
   - 避免命令式語法可能帶來的副作用

## 常見的條件邏輯模式

### 1. 三元運算子（最常用）

```hcl
# 替代 if-else
count = var.environment == "prod" ? 1 : 0

# 替代 if-elseif-else
name = var.environment == "prod" ? "prod-cluster" : 
       var.environment == "staging" ? "staging-cluster" : 
       "dev-cluster"
```

### 2. count 條件建立

```hcl
# 替代 if condition: create resource
resource "aws_instance" "conditional" {
  count = var.create_instance ? 1 : 0
  # 資源配置...
}
```

### 3. for_each 動態建立

```hcl
# 替代 for loop
resource "aws_instance" "multiple" {
  for_each = var.instance_configs
  # 資源配置...
}
```

### 4. dynamic 區塊

```hcl
# 替代巢狀 if-else
resource "aws_instance" "example" {
  dynamic "ebs_block_device" {
    for_each = var.create_ebs ? [1] : []
    content {
      # 區塊內容...
    }
  }
}
```

## 實際應用範例

### 錯誤處理機制

```hcl
# 驗證 VPC 配置
resource "null_resource" "vpc_config_validation" {
  count = local.vpc_config_valid ? 0 : 1

  provisioner "local-exec" {
    command = "echo 'Error: Invalid VPC configuration. Please ensure networking module is deployed and vpc_config is properly configured.' && exit 1"
  }
}
```

### 環境特定配置

```hcl
resource "aws_eks_cluster" "main" {
  count = var.environment == "prod" ? 1 : 
          var.environment == "staging" ? 1 : 1
  
  name = var.environment == "prod" ? "prod-cluster" :
         var.environment == "staging" ? "staging-cluster" :
         "dev-cluster"
}
```

## 最佳實踐

1. **使用三元運算子** 進行簡單的條件判斷
2. **使用 count** 進行條件資源建立
3. **使用 for_each** 進行動態資源建立
4. **使用 dynamic 區塊** 處理複雜的條件邏輯
5. **保持程式碼可讀性**，適當添加註解說明複雜的條件邏輯

## 總結

Terraform 的條件邏輯設計反映了其宣告式語言的本質：
- 專注於描述「要什麼」而非「如何做」
- 透過條件運算子和動態區塊實現複雜邏輯
- 確保基礎設施配置的可預測性和一致性

這種設計讓 Terraform 更適合管理複雜的基礎設施配置，同時保持程式碼的可讀性和可維護性。
