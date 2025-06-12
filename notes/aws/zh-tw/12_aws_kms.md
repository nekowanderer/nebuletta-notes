# 關於 AWS KMS (Key Management Service)

[English](../en/12_aws_kms.md) | [繁體中文](./12_aws_kms.md) | [日本語](../ja/12_aws_kms.md) | [回到索引](../README.md)

### 什麼是 KMS？

AWS KMS 是一個全託管的加密金鑰管理服務，用於建立和控制加密金鑰，以加密您的資料。KMS 與其他 AWS 服務整合，讓您能夠輕鬆加密儲存在這些服務中的資料。

### 主要特性

#### 1. 金鑰管理
- 可以建立、導入、輪換、刪除和管理加密金鑰
- 支援對稱和非對稱金鑰
- 自動進行金鑰備份和版本控制
- 提供金鑰別名功能，方便管理

#### 2. 加密操作
- 提供加密和解密 API
- 支援多種加密演算法
- 可以對資料進行加密，也可以生成資料金鑰
- 支援跨區域金鑰複製

#### 3. 安全性
- 金鑰永遠不會離開 KMS 服務
- 提供硬體安全模組 (HSM) 選項
- 符合多種安全標準和法規要求
- 支援 CloudTrail 審計日誌

### 使用場景

#### 1. 資料加密
- S3 物件加密
- DynamoDB 表格加密
- EBS 卷加密
- RDS 資料庫加密
- Cognito user pool 加密

#### 2. 應用程式加密
- 應用程式資料加密
- 敏感資訊保護
- 跨服務資料加密

#### 3. 合規要求
- 資料保護法規
- 安全標準合規
- 審計追蹤

### AWS 資源的 KMS 政策

#### 1. 為什麼需要 KMS 政策？
- 控制哪些 AWS 服務可以使用 KMS 金鑰
- 確保只有授權的服務可以進行加密/解密操作
- 符合最小權限原則

#### 2. 常見服務的 KMS 政策範例

#### S3 服務
```json
{
  "Sid": "AllowS3UseKey",
  "Effect": "Allow",
  "Principal": {
    "Service": "s3.amazonaws.com"
  },
  "Action": [
    "kms:Encrypt",
    "kms:Decrypt",
    "kms:ReEncrypt*",
    "kms:GenerateDataKey*",
    "kms:DescribeKey"
  ],
  "Resource": "*"
}
```

#### DynamoDB 服務
```json
{
  "Sid": "AllowDynamoDBUseKey",
  "Effect": "Allow",
  "Principal": {
    "Service": "dynamodb.amazonaws.com"
  },
  "Action": [
    "kms:Encrypt",
    "kms:Decrypt",
    "kms:ReEncrypt*",
    "kms:GenerateDataKey*",
    "kms:DescribeKey"
  ],
  "Resource": "*"
}
```

#### 3. 政策設定注意事項
- 只給予必要的權限
- 使用服務特定的 Principal
- 定期審計政策設定
- 監控金鑰使用情況

### IAM 角色的 KMS 政策

#### 1. 為什麼需要 IAM 角色的 KMS 政策？
- 控制哪些用戶/角色可以使用 KMS 金鑰
- 管理加密/解密操作的權限
- 實現細粒度的存取控制

#### 2. 常見的 IAM 角色政策範例

##### 管理員角色
```json
{
  "Sid": "AllowAccountRoot",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::ACCOUNT_ID:root"
  },
  "Action": [
    "kms:*"  # 完整的管理權限
  ],
  "Resource": "*"
}
```

##### 開發者角色
```json
{
  "Sid": "AllowDeveloperAccess",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::ACCOUNT_ID:role/DeveloperRole"
  },
  "Action": [
    "kms:Encrypt",
    "kms:Decrypt",
    "kms:ReEncrypt",
    "kms:GenerateDataKey",
    "kms:DescribeKey"
  ],
  "Resource": "*"
}
```

##### SSO 用戶角色
```hcl
{
  "Sid": "AllowSSOUserAccess",
  "Effect": "Allow",
  "Principal": {
    "AWS": [
      "arn:aws:sts::ACCOUNT_ID:assumed-role/SSO_ROLE/user1",
      "arn:aws:sts::ACCOUNT_ID:assumed-role/SSO_ROLE/user2"
    ]
  },
  "Action": [
    "kms:Encrypt",
    "kms:Decrypt",
    "kms:ReEncrypt",
    "kms:GenerateDataKey",
    "kms:DescribeKey"
  ],
  "Resource": "*"
}
```

#### 3. IAM 角色政策最佳實踐

##### 權限分級
- 管理員：完整權限（kms:*）
- 開發者：基本操作權限
- 一般用戶：有限的操作權限

##### 安全考慮
- 使用最小權限原則
- 定期審計權限設定
- 監控異常使用情況
- 實施權限分離

##### 政策管理
- 使用群組管理權限
- 定期輪換金鑰
- 記錄所有金鑰操作
- 設定適當的警報

#### 4. 常見使用場景

##### 應用程式加密
- 應用程式需要加密敏感資料
- 使用 IAM 角色存取 KMS
- 實作加密/解密邏輯

##### 跨服務加密
- 多個服務使用同一個 KMS 金鑰
- 透過 IAM 角色控制存取
- 確保資料一致性

##### 合規要求
- 符合資料保護法規
- 實作存取控制
- 提供審計追蹤

### KMS 金鑰的使用方式

#### 1. 直接使用 vs 間接使用

##### 直接使用（不建議）
- 直接透過 IAM 角色/用戶存取 KMS API
- 需要手動處理加密/解密邏輯
- 需要管理金鑰生命週期
- 容易造成安全風險

##### 間接使用（建議）
- 透過 AWS 服務自動處理加密/解密
- 例如：
  - S3 自動加密物件
  - DynamoDB 自動加密資料
  - EBS 自動加密磁碟
- 更安全且易於管理

#### 2. 為什麼建議間接使用？

##### 安全性
- 金鑰永遠不會離開 KMS 服務
- 減少人為錯誤
- 降低安全風險
- 符合最小權限原則

##### 便利性
- 自動處理加密/解密
- 不需要額外的程式碼
- 減少維護成本
- 更容易符合合規要求

##### 成本效益
- 減少 API 調用次數
- 降低操作錯誤風險
- 簡化監控和審計
- 更好的資源利用

### 成本考量

#### 1. 金鑰費用
- 客戶管理的金鑰（CMK）：每月 $1/個
- AWS 管理的金鑰：免費

#### 2. API 調用費用
- 每個 API 調用 $0.03
- 包括加密、解密、生成資料金鑰等操作

#### 3. 成本優化建議
- 使用 AWS 管理的金鑰（如果可能）
- 合併使用同一個金鑰
- 避免不必要的金鑰輪換
- 監控 API 調用次數

### 最佳實踐

#### 1. 金鑰管理
- 使用金鑰別名而不是金鑰 ID
- 定期輪換金鑰
- 使用適當的 IAM 權限
- 啟用 CloudTrail 日誌

#### 2. 安全性
- 遵循最小權限原則
- 定期審計金鑰使用情況
- 監控異常活動
- 使用適當的加密演算法

#### 3. 成本控制
- 選擇合適的金鑰類型
- 優化 API 調用
- 定期檢查使用情況
- 移除未使用的金鑰

### 限制

#### 1. 技術限制
- 每個金鑰的加密資料大小限制
- API 調用限制
- 金鑰數量限制

#### 2. 成本限制
- 金鑰費用
- API 調用費用
- 資料傳輸費用

### 總結

AWS KMS 提供了一個安全、可靠且易於使用的金鑰管理解決方案。它與其他 AWS 服務無縫整合，讓您能夠輕鬆實現資料加密需求。雖然會產生一些費用，但考慮到其提供的安全性和便利性，這些成本是值得的。在實際使用中，應該根據具體需求選擇合適的金鑰類型和管理策略，並遵循最佳實踐來確保安全性和成本效益。
