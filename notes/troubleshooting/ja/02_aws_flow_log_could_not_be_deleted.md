# AWS Flow Log が削除できない問題のトラブルシューティング

[English](../en/02_aws_flow_log_could_not_be_deleted.md) | [繁體中文](../zh-tw/02_aws_flow_log_could_not_be_deleted.md) | [日本語](02_aws_flow_log_could_not_be_deleted.md) | [インデックスに戻る](../README.md)

---

## 背景
- 実験日: 2025/05/24
- 難易度：🤬🤬
- 説明: `terraform destroy` を実行した際、AWS Flow Log と関連する CloudWatch Log Group が正しく削除されず、Terraform が "Destruction complete" と表示したにもかかわらず、リソースが残っている状態。

## 発生した問題

- `terraform destroy` は正常に実行され、リソースが削除されたと表示
- AWS Console で CloudWatch Log Group が依然として表示される
- AWS CLI で手動確認したところ、リソースが実際に存在していることを確認：
  ```bash
  aws logs describe-log-groups --region ap-northeast-1 --log-group-name-prefix "/aws/dev-infra-networking"
  ```

## デバッグプロセス

### 第一段階：問題範囲の確認

以下の項目を確認：

| 項目 | 状態 | 説明 |
|------|------|------|
| VPC Flow Log | ✅ | Terraform で削除済みと表示 |
| CloudWatch Log Group | ❌ | AWS Console に依然として存在 |
| IAM Role | ✅ | 正しく削除済み |
| VPC | ✅ | 正しく削除済み |

### 第二段階：問題の調査

1. GitHub Issues を確認
2. AWS Provider の既知の問題であることを発見：[Issue #34996](https://github.com/hashicorp/terraform-provider-aws/issues/34996)
3. 影響範囲：
   - `aws_cloudwatch_log_group` リソース
   - `skip_destroy = false` でも発生
     - 注：この構文は `v1.12.0` 以降で非推奨
   - Terraform Cloud でも発生

### 第三段階：解決策の設計

`null_resource` を一時的な解決策として使用：

1. ✅ `triggers` で必要な情報を保存
2. ✅ `local-exec` provisioner を使用
3. ✅ `destroy` フェーズでクリーンアップスクリプトを実行
4. ✅ AWS CLI で残存リソースを手動削除

## 根本原因

### AWS Provider の制限

1. **Provider の動作**：
   - Terraform はリソースが削除されたと表示
   - しかし実際の AWS リソースは残存
   - これは Provider の既知の問題

2. **影響範囲**：
   - `aws_cloudwatch_log_group` リソースに影響
   - 特に VPC Flow Log 関連のシナリオで発生
   - Terraform Cloud でも問題が発生

## 解決策

### null_resource を一時的な解決策として使用

```hcl
resource "null_resource" "delete_flow_log" {
  triggers = {
    prefix = local.prefix
  }

  depends_on = [aws_flow_log.vpc_flow_log]

  provisioner "local-exec" {
    when    = destroy
    command = <<-EOT
      # VPC Flow Log の削除を待機
      echo "Waiting for VPC Flow Log to be deleted..."
      sleep 10

      # VPC Flow Log が存在するか確認
      FLOW_LOG_ID=$(aws ec2 describe-flow-logs --region ap-northeast-1 --query "FlowLogs[?LogGroupName=='/aws/${self.triggers.prefix}/vpc/flow-logs'].FlowLogId" --output text)
      
      if [ ! -z "$FLOW_LOG_ID" ]; then
        echo "VPC Flow Log still exists, deleting manually..."
        aws ec2 delete-flow-logs --flow-log-ids $FLOW_LOG_ID --region ap-northeast-1
      fi

      # CloudWatch Log Group が存在するか確認
      LOG_GROUP_NAME="/aws/${self.triggers.prefix}/vpc/flow-logs"
      if aws logs describe-log-groups --log-group-name-prefix "$LOG_GROUP_NAME" --region ap-northeast-1 --query "logGroups[?logGroupName=='$LOG_GROUP_NAME']" --output text | grep -q "$LOG_GROUP_NAME"; then
        echo "CloudWatch Log Group still exists, deleting manually..."
        aws logs delete-log-group --log-group-name "$LOG_GROUP_NAME" --region ap-northeast-1
      fi
    EOT
  }
}
```

## 重要な学び

### null_resource の特性

| 特性 | 説明 |
|------|------|
| 仮想リソース | AWS に実際のリソースを作成しない |
| ライフサイクル | create、update、destroy フェーズで操作を実行可能 |
| トリガー条件 | triggers で状態を保存可能 |
| 実行タイミング | when パラメータで実行タイミングを制御可能 |

### provisioner の使用

| タイプ | 用途 | 適用シナリオ |
|--------|------|--------------|
| local-exec | ローカルコマンドの実行 | リソースのクリーンアップ、スクリプト実行 |
| remote-exec | リモートコマンドの実行 | リモートホストの設定 |
| file | ファイル転送 | 設定ファイルのデプロイ |

### 解決策のメリット・デメリット

メリット：
- クリーンアッププロセスの自動化
- Provider の削除メカニズムに依存しない
- リソースが確実に削除される

デメリット：
- 追加の IAM 権限が必要：
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
- Terraform 設定の複雑さが増加
- 一時的な解決策

## 予防措置

1. **Provider の更新を監視**：
   - [GitHub Issue #34996](https://github.com/hashicorp/terraform-provider-aws/issues/34996) をフォロー
   - 公式の修正を待つ

2. **定期的なリソースチェック**：
   - AWS CLI でリソース状態を確認
   - リソースが確実に削除されていることを確認

3. **クリーンアッププロセスの記録**：
   - クリーンアップ操作をログに記録
   - 追跡とデバッグを容易に

## 結論

このデバッグ経験から、Terraform で AWS リソースを管理する際は、Provider の制限や既知の問題に特に注意を払う必要があることを学びました。`null_resource` は最もエレガントな解決策ではありませんが、リソースが確実に削除されるための効果的な一時的な解決策を提供します。

**注意**: Terraform が "Destruction complete" と表示しても、特に CloudWatch Log Group を扱う場合は、リソースが実際に削除されたとは限りません！ 