# Terraform 狀態管理

[English](../en/README.md) | [繁體中文](README.md) | [日本語](../ja/README.md) | [回到索引](../README.md)

此模組用於創建和管理遠端 Terraform 狀態存儲。它創建：
- 用於存儲 Terraform 狀態文件的 S3 儲存桶
- 用於狀態鎖定的 DynamoDB 表格

## 前置需求

- 已安裝並配置 AWS CLI
- 已完成 AWS SSO 登入
- 使用具有足夠權限的設定檔

## 使用方法

1. 初始化 Terraform：
```bash
terraform init
```

2. 執行規劃：
```bash
terraform plan -var-file="common.tfvars"
```

3. 應用變更：
```bash
terraform apply -var-file="common.tfvars"
```

4. 銷毀資源（如需要）：
```bash
terraform destroy -var-file="common.tfvars"
```

## 變數

| 變數名稱 | 描述 | 範例 |
|--------------|-------------|---------|
| env | 環境名稱 | dev |
| module_name | 模組名稱 | infra |
| aws_region | AWS 區域 | ap-northeast-1 |

## 輸出

| 輸出名稱 | 描述 |
|------------|-------------|
| storage_name | S3 儲存桶名稱 |
| storage_arn | S3 儲存桶 ARN |
| lock_table_name | DynamoDB 表格名稱 |
| lock_table_arn | DynamoDB 表格 ARN |

## 注意事項

1. 此模組創建的資源僅用於 Terraform 狀態存儲
2. 確保使用正確的 AWS 設定檔
3. 在開發環境中建議不要設置 `prevent_destroy`
4. Remote state storage 本身的 tfstate 通常是不會放S3的，他要做的就只是在 AWS 上面開一個空的 bucket 還有一張 DynamoDB table 當作 lock 而已
