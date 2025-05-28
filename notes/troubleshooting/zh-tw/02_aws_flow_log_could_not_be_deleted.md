# AWS Flow Log 無法被正確刪除的除錯經驗

[English](../en/02_aws_flow_log_could_not_be_deleted.md) | [繁體中文](02_aws_flow_log_could_not_be_deleted.md) | [日本語](../ja/02_aws_flow_log_could_not_be_deleted.md) | [返回索引](../README.md)

---

## 背景
- 實驗日期: 2025/05/24
- 難度：🤬🤬
- 描述: 在執行 `terraform destroy` 時，AWS Flow Log 和相關的 CloudWatch Log Group 無法被正確刪除，即使 Terraform 顯示 "Destruction complete"。

## 遇到的現象

- `terraform destroy` 執行成功，顯示資源已被刪除
- AWS Console 中仍然可以看到 CloudWatch Log Group
- 手動檢查 AWS CLI 確認資源確實還存在：
  ```bash
  aws logs describe-log-groups --region ap-northeast-1 --log-group-name-prefix "/aws/dev-infra-networking"
  ```

## 除錯過程

### 第一階段：確認問題範圍

已確認以下各項目：

| 項目 | 狀態 | 說明 |
|------|------|------|
| VPC Flow Log | ✅ | Terraform 顯示已刪除 |
| CloudWatch Log Group | ❌ | 仍然存在於 AWS Console |
| IAM Role | ✅ | 已正確刪除 |
| VPC | ✅ | 已正確刪除 |

### 第二階段：研究問題

1. 檢查 GitHub Issues
2. 發現這是 AWS Provider 的已知問題：[Issue #34996](https://github.com/hashicorp/terraform-provider-aws/issues/34996)
3. 問題影響：
   - `aws_cloudwatch_log_group` 資源
   - 即使 `skip_destroy = false`
     - 然後這個語法升新版 (`v1.12.0`) 後被 deprecate 了
   - 在 Terraform Cloud 中也會發生

### 第三階段：解決方案設計

使用 `null_resource` 作為臨時解決方案：

1. ✅ 使用 `triggers` 保存必要資訊
2. ✅ 使用 `local-exec` provisioner
3. ✅ 在 `destroy` 階段執行清理腳本
4. ✅ 使用 AWS CLI 手動刪除殘留資源

## 問題根因

### AWS Provider 的限制

1. **Provider 行為**：
   - Terraform 顯示資源已刪除
   - 但實際 AWS 資源仍然存在
   - 這是一個已知的 Provider 問題

2. **影響範圍**：
   - 影響 `aws_cloudwatch_log_group` 資源
   - 特別是在與 VPC Flow Log 相關的場景
   - 問題在 Terraform Cloud 中也會發生

## 解決方案

### 使用 null_resource 作為臨時解決方案

```hcl
resource "null_resource" "delete_flow_log" {
  triggers = {
    prefix = local.prefix
  }

  depends_on = [aws_flow_log.vpc_flow_log]

  provisioner "local-exec" {
    when    = destroy
    command = <<-EOT
      # 等待 VPC Flow Log 被刪除
      echo "Waiting for VPC Flow Log to be deleted..."
      sleep 10

      # 檢查 VPC Flow Log 是否還存在
      FLOW_LOG_ID=$(aws ec2 describe-flow-logs --region ap-northeast-1 --query "FlowLogs[?LogGroupName=='/aws/${self.triggers.prefix}/vpc/flow-logs'].FlowLogId" --output text)
      
      if [ ! -z "$FLOW_LOG_ID" ]; then
        echo "VPC Flow Log still exists, deleting manually..."
        aws ec2 delete-flow-logs --flow-log-ids $FLOW_LOG_ID --region ap-northeast-1
      fi

      # 檢查 CloudWatch Log Group 是否還存在
      LOG_GROUP_NAME="/aws/${self.triggers.prefix}/vpc/flow-logs"
      if aws logs describe-log-groups --log-group-name-prefix "$LOG_GROUP_NAME" --region ap-northeast-1 --query "logGroups[?logGroupName=='$LOG_GROUP_NAME']" --output text | grep -q "$LOG_GROUP_NAME"; then
        echo "CloudWatch Log Group still exists, deleting manually..."
        aws logs delete-log-group --log-group-name "$LOG_GROUP_NAME" --region ap-northeast-1
      fi
    EOT
  }
}
```

## 關鍵學習重點

### null_resource 的特性

| 特性 | 說明 |
|------|------|
| 虛擬資源 | 不會在 AWS 中創建實際資源 |
| 生命週期 | 可以執行 create、update、destroy 階段的操作 |
| 觸發條件 | 可以通過 triggers 保存狀態 |
| 執行時機 | 可以通過 when 參數控制執行時機 |

### provisioner 的使用

| 類型 | 用途 | 適用場景 |
|------|------|----------|
| local-exec | 執行本地命令 | 清理資源、執行腳本 |
| remote-exec | 執行遠端命令 | 配置遠端主機 |
| file | 傳輸檔案 | 部署配置文件 |

### 解決方案的優缺點

優點：
- 自動化清理過程
- 不依賴 Provider 的刪除機制
- 可以確保資源被正確刪除

缺點：
- 需要額外的 IAM 權限，以這個案例來說就是：
  ```json
  {
      "Version": "2012-10-17",
      "Statement": [
          {
              "Effect": "Allow",
              "Action": [
                  "ec2:DeleteFlowLogs",
                  "logs:DeleteLogGroup"
              ],
              "Resource": "*"
          }
      ]
  }
  ```
- 增加 Terraform 配置的複雜度
- 是臨時解決方案

## 預防措施

1. **監控 Provider 更新**：
   - 關注 [GitHub Issue #34996](https://github.com/hashicorp/terraform-provider-aws/issues/34996)
   - 等待官方修復

2. **定期檢查資源**：
   - 使用 AWS CLI 檢查資源狀態
   - 確保資源被正確刪除

3. **記錄清理過程**：
   - 在日誌中記錄清理操作
   - 方便追蹤和除錯

## 結論

這次的除錯經驗提醒我們，在使用 Terraform 管理 AWS 資源時，需要特別注意 Provider 的限制和已知問題。雖然 `null_resource` 不是最優雅的解決方案，但它提供了一個有效的臨時解決方案，確保資源能夠被正確刪除。

**記住**：Terraform 顯示 "Destruction complete" 不代表資源真的被刪除了，特別是在處理 CloudWatch Log Group 時！ 