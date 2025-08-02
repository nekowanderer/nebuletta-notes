# Terraform State Lock復旧ガイド

[English](../en/06_terraform_state_lock_recovery_guide.md) | [繁體中文](../zh-tw/06_terraform_state_lock_recovery_guide.md) | [日本語](../ja/06_terraform_state_lock_recovery_guide.md) | [インデックスに戻る](../README.md)

---

## 背景
- 実験日：2025/08/02
- 難易度：🤬
- 説明：EKSクラスター作成中にAWS SSOセッションタイムアウトによりエラーが発生。

---

## 問題の説明

Terraformの操作中にAWSトークンが期限切れになると、以下のエラーが発生する可能性があります：

### エラーメッセージの例

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

## 根本原因

1. **AWSトークンの期限切れ**: 長時間のデプロイプロセス中にAWSセッショントークンが期限切れ
2. **State Lockの固着**: DynamoDB内のTerraform state lockが正常に解放されない
3. **Stateファイルの分離**: Terraformが現在の状態をローカルの`errored.tfstate`ファイルに書き込み

---

## 解決手順

### ステップ1：State Lockの強制解除

エラーメッセージからlock IDを見つけます。例：`d27f3f1c-6ce8-c97b-0a3f-e6f77255b429`

```bash
# 構文：terraform force-unlock <LOCK_ID>
$ terraform force-unlock d27f3f1c-6ce8-c97b-0a3f-e6f77255b429
```

システムが確認を要求します：
```
Do you really want to force-unlock?
  Terraform will remove the lock on the remote state.
  This will allow local Terraform commands to modify this state, even though it
  may still be in use. Only 'yes' will be accepted to confirm.

  Enter a value: yes
```

`yes`と入力して確認します。

**⚠️ 警告**: これは危険な操作です。同じstateで他の人が作業していないことが確実な場合のみ実行してください。

### ステップ2：AWS認証の確認と更新

```bash
# 現在のAWS認証状態を確認
$ aws sts get-caller-identity
```

認証期限切れエラーが表示される場合、再認証が必要です：
```bash
# AWS SSOログイン（設定に応じて）
$ aws sso login --profile your-profile

# または他の認証方法を使用
$ aws configure
```

### ステップ3：Stateファイルのプッシュ

ローカルの`errored.tfstate`をリモートバックエンドにプッシュします：

```bash
$ terraform state push errored.tfstate
```

成功後、ローカルのエラーファイルを削除できます：
```bash
$ rm errored.tfstate
```

### ステップ4：状態復旧の検証

```bash
# state状態を確認
$ terraform state list

# プランを表示して異常がないことを確認
$ terraform plan
```

---

## 技術的原理

### Terraform State Lockメカニズム

- **目的**: 複数人が同じインフラストラクチャを同時に変更することを防ぐ
- **実装**: DynamoDBテーブルを使用してロック情報を保存
- **問題**: 操作が異常終了した場合、ロックが正しく解放されない可能性

### Stateファイル管理

- **リモートバックエンド**: 本番環境のstateはS3に保存され、DynamoDBでロック
- **ローカルバックアップ**: 操作が失敗した場合、Terraformは現在の状態をローカルの`errored.tfstate`に保存
- **復旧**: `terraform state push`を使用してローカル状態をリモートにプッシュ

### AWSトークン期限切れの処理

- **原因**: 長時間の操作がトークンの有効期限を超える可能性
- **影響**: S3バックエンドとDynamoDBロックテーブルにアクセスできない
- **解決**: 再認証後に操作を継続

---

## 予防措置

1. **トークン有効期限の監視**: 長時間の操作前にトークンの残り時間を確認
2. **段階的デプロイ**: 大規模インフラストラクチャを複数の小さなモジュールに分割
3. **長期資格情報の使用**: セキュリティが許可する場合、より長い有効期限の認証方式を使用
4. **定期的なロック確認**: 操作前に残留するstate lockがないか確認

---

## 関連コマンドリファレンス

```bash
# 現在のstate lock状態を表示
$ terraform providers lock

# state内のすべてのリソースをリスト
$ terraform state list

# 特定リソースのstateを表示
$ terraform state show <resource_name>

# 既存リソースをstateにインポート
$ terraform import <resource_type>.<resource_name> <resource_id>

# stateからリソースを削除（実際のリソースは削除しない）
$ terraform state rm <resource_name>
```

---

## トラブルシューティング

上記の手順で問題が解決しない場合：

1. **AWS権限の確認**: S3バケットとDynamoDBテーブルにアクセスする十分な権限があることを確認
2. **ネットワーク接続の確認**: AWSサービスに正常にアクセスできることを確認
3. **バックエンド設定の確認**: `backend.tf`設定が正しいことを確認
4. **チームへの連絡**: 共有環境の場合、他の人が操作していないことを確認