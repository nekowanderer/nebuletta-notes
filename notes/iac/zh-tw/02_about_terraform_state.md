# 關於 Terraform 狀態

[English](../en/02_about_terraform_state.md) | [繁體中文](02_about_terraform_state.md) | [日本語](../ja/02_about_terraform_state.md) | [回到索引](../README.md)

## 敏感資訊處理

- ARN、ID 和其他敏感資訊不應出現在 `.tf` 或 `.hcl` 文件中
- 這些敏感數據存儲在 `.tfstate` 文件中
- `.tfstate` 文件應添加到 `.gitignore`

## 狀態文件生命週期

### 在 `terraform plan` 期間：
- Terraform 讀取遠程狀態
- 比較本地代碼與遠程狀態
- 生成變更計劃

### 在 `terraform apply` 期間：
- 應用變更
- 更新遠程狀態
- 本地狀態與遠程同步

## 後端配置

### 什麼是 `backend.hcl`？
- 一個定義 Terraform 狀態存儲位置和方式的配置文件
- 包含靜態後端設置，如儲存桶名稱、區域等
- 不能使用變數或引用其他文件
- 應保持簡單和靜態

### 使用 `-backend-config`：
- 允許在初始化期間傳遞後端配置
- 可用於為不同環境提供不同的後端設置
- 例如：`terraform init -backend-config=../state-storage/backend.hcl`
- 當您需要在多個模組中使用相同的後端配置時很有用

## 最佳實踐

1. **版本控制**：
   - `.tfstate` 文件不應被版本控制
   - 使用遠程狀態存儲（例如 S3）
   - 使用狀態鎖定（例如 DynamoDB）
   - 敏感信息僅保存在狀態中

2. **為什麼採用這種方法**：
   - **安全性**：防止敏感信息洩露
   - **協作**：實現安全的多用戶基礎設施管理
   - **追蹤**：允許追蹤基礎設施變更
   - **備份**：狀態文件安全地備份在 S3 中

## 當前設置

本庫的當前配置遵循這些最佳實踐：
- 使用 S3 進行狀態存儲
- 使用 DynamoDB 進行狀態鎖定
- 敏感信息僅存在於狀態中
- 代碼僅包含配置和邏輯 