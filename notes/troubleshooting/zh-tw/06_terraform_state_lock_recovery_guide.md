# Terraform State Lock Recovery Guide

[English](../en/06_terraform_state_lock_recovery_guide.md) | [繁體中文](../zh-tw/06_terraform_state_lock_recovery_guide.md) | [日本語](../ja/06_terraform_state_lock_recovery_guide.md) | [回到索引](../README.md)

---

## 背景
- 實驗日期：2025/08/02
- 難度：🤬
- 描述：建立 EKS cluster 的途中出現錯誤訊息，原因是 AWS SSO session timeoiut。

---

## 問題描述

當 Terraform 操作過程中 AWS token 過期時，可能會導致以下錯誤：

### 錯誤訊息範例

```
│ Error: failed to upload state: operation error S3: PutObject, https response error StatusCode: 400, RequestID: FGTNVZ1CHFZ0N23K, HostID: LJbubun1tq2Ezh0AoRHi0kouNbISTJ2xXWUGtp6Du4imCnwtr8DQZtU1K4ARw2L6UwsF/pPbGg77TulnSrg//xXFtmm432ZZeSQIOdiVPZk=, api error ExpiredToken: The provided token has expired.
│
│
╵
╷
│ Error: Failed to save state
│
│ Error saving state: failed to upload state: operation error S3: PutObject, https response error StatusCode: 400, RequestID: FGTNYRRQXYFHRPNX, HostID:
│ kWV7p03zXxLXGWK2NIMuCaxTvHbYYfY4EezTNeMigRFZsdjyzrHh6qvm4qUSx3rH6b7bwAivwoweHZAiwCVtO5d1pm7/fpmE892YyZQca7Y=, api error ExpiredToken: The provided token has expired.
╵
╷
│ Error: Failed to persist state to backend
│
│ The error shown above has prevented Terraform from writing the updated state to the configured backend. To allow for recovery, the state has been written to the file "errored.tfstate"
│ in the current working directory.
│
│ Running "terraform apply" again at this point will create a forked state, making it harder to recover.
│
│ To retry writing this state, use the following command:
│     terraform state push errored.tfstate
│
╵
╷
│ Error: waiting for EKS Cluster (dev-eks-cluster) create: operation error EKS: DescribeCluster, https response error StatusCode: 403, RequestID: 30e246f7-385f-4274-b154-c6bbae9fd4cf, api error ExpiredTokenException: The security token included in the request is expired
│
│   with module.eks_cluster.aws_eks_cluster.main,
│   on ../../../../modules/eks-lab/cluster/eks_cluster.tf line 13, in resource "aws_eks_cluster" "main":
│   13: resource "aws_eks_cluster" "main" {
│
╵
╷
│ Error: Error releasing the state lock
│
│ Error message: failed to retrieve lock info for lock ID "d27f3f1c-6ce8-c97b-0a3f-e6f77255b429": Unable to retrieve item from DynamoDB table "dev-state-storage-locks": operation error
│ DynamoDB: GetItem, https response error StatusCode: 400, RequestID: 60S67NR6MAIKT5TH73QBL5AHCVVV4KQNSO5AEMVJF66Q9ASUAAJG, api error ExpiredTokenException: The security token included in
│ the request is expired
│
│ Terraform acquires a lock when accessing your state to prevent others
│ running Terraform to potentially modify the state at the same time. An
│ error occurred while releasing this lock. This could mean that the lock
│ did or did not release properly. If the lock didn't release properly,
│ Terraform may not be able to run future commands since it'll appear as if
│ the lock is held.
│
│ In this scenario, please call the "force-unlock" command to unlock the
│ state manually. This is a very dangerous operation since if it is done
│ erroneously it could result in two people modifying state at the same time.
│ Only call this command if you're certain that the unlock above failed and
│ that no one else is holding a lock.
╵
```

---

## 問題原因

1. **AWS Token 過期**：長時間部署過程中 AWS session token 過期
2. **State Lock 卡住**：DynamoDB 中的 Terraform state lock 無法正常釋放
3. **State 檔案分離**：Terraform 將當前狀態寫入本地 `errored.tfstate` 檔案

---

## 解決步驟

### 步驟一：強制解鎖 State Lock

從錯誤訊息中找到 lock ID，例如：`d27f3f1c-6ce8-c97b-0a3f-e6f77255b429`

```bash
# 語法：terraform force-unlock <LOCK_ID>
terraform force-unlock d27f3f1c-6ce8-c97b-0a3f-e6f77255b429
```

系統會要求確認：
```
Do you really want to force-unlock?
  Terraform will remove the lock on the remote state.
  This will allow local Terraform commands to modify this state, even though it
  may still be in use. Only 'yes' will be accepted to confirm.

  Enter a value: yes
```

輸入 `yes` 確認。

**⚠️ 警告**：這是危險操作，只有在確定沒有其他人在操作同一個 state 時才執行。

### 步驟二：檢查並更新 AWS 認證

```bash
# 檢查目前的 AWS 認證狀態
aws sts get-caller-identity
```

如果顯示認證過期錯誤，需要重新認證：
```bash
# AWS SSO 登入（依據你的設定方式）
aws sso login --profile your-profile

# 或使用其他認證方式
aws configure
```

### 步驟三：推送 State 檔案

將本機的 `errored.tfstate` 推送回遠端 backend：

```bash
terraform state push errored.tfstate
```

成功後可以刪除本機的錯誤檔案：
```bash
rm errored.tfstate
```

### 步驟四：驗證狀態恢復

```bash
# 檢查 state 狀態
terraform state list

# 查看計劃，確認沒有異常
terraform plan
```

---

## 技術原理

### Terraform State Lock 機制

- **用途**：防止多人同時修改同一個 infrastructure
- **實作**：使用 DynamoDB 表格存儲 lock 信息
- **問題**：當操作異常中斷時，lock 可能沒有正確釋放

### State 檔案管理

- **Remote Backend**：生產環境的 state 存在 S3，由 DynamoDB 鎖定
- **Local Backup**：操作失敗時，Terraform 會將當前狀態保存到本地 `errored.tfstate`
- **Recovery**：透過 `terraform state push` 將本地狀態推送回遠端

### AWS Token 過期處理

- **原因**：長時間操作可能超過 token 有效期
- **影響**：無法存取 S3 backend 和 DynamoDB lock table
- **解決**：重新認證後繼續操作

---

## 預防措施

1. **監控 Token 有效期**：長時間操作前確認 token 剩餘時間
2. **分階段部署**：將大型 infrastructure 分成多個較小的模組
3. **使用長期憑證**：在安全允許的情況下使用較長有效期的認證方式
4. **定期檢查 Lock**：操作前檢查是否有殘餘的 state lock

---

## 相關指令參考

```bash
# 查看目前 state lock 狀態
terraform providers lock

# 列出所有 state 中的資源
terraform state list

# 查看特定資源的 state
terraform state show <resource_name>

# 匯入現有資源到 state
terraform import <resource_type>.<resource_name> <resource_id>

# 移除 state 中的資源（不刪除實際資源）
terraform state rm <resource_name>
```

---

## 故障排除

如果上述步驟無法解決問題：

1. **檢查 AWS 權限**：確認有足夠權限存取 S3 bucket 和 DynamoDB table
2. **檢查網路連線**：確認可以正常存取 AWS 服務
3. **檢查 Backend 設定**：確認 `backend.tf` 設定正確
4. **聯絡團隊**：如果是共享環境，確認沒有其他人在操作
