# Terraform 状態について

[English](../en/02_about_terraform_state.md) | [繁體中文](../zh-tw/02_about_terraform_state.md) | [日本語](02_about_terraform_state.md) | [索引に戻る](../README.md)

## 機密情報の取り扱い

- ARN、ID、その他の機密情報は `.tf` または `.hcl` ファイルに記載すべきではありません
- これらの機密データは `.tfstate` ファイルに保存されます
- `.tfstate` ファイルは `.gitignore` に追加すべきです

## 状態ファイルのライフサイクル

### `terraform plan` 実行中：
- Terraform はリモート状態を読み取ります
- ローカルコードとリモート状態を比較します
- 変更計画を生成します

### `terraform apply` 実行中：
- 変更を適用します
- リモート状態を更新します
- ローカル状態をリモートと同期します

## バックエンド構成

### `backend.hcl` とは？
- Terraform 状態の保存場所と方法を定義する構成ファイル
- バケット名、リージョンなどの静的バックエンド設定を含みます
- 変数を使用したり、他のファイルを参照したりすることはできません
- シンプルで静的に保つべきです

### `-backend-config` の使用：
- 初期化中にバックエンド構成を渡すことができます
- 異なる環境に異なるバックエンド設定を提供するために使用できます
- 例：`terraform init -backend-config=../state-storage/backend.hcl`
- 複数のモジュールで同じバックエンド構成を使用する必要がある場合に役立ちます

## ベストプラクティス

1. **バージョン管理**：
   - `.tfstate` ファイルはバージョン管理すべきではありません
   - リモート状態ストレージを使用する（例：S3）
   - 状態ロックを使用する（例：DynamoDB）
   - 機密情報は状態内のみに保持する

2. **このアプローチを採用する理由**：
   - **セキュリティ**：機密情報の露出を防ぎます
   - **コラボレーション**：安全なマルチユーザーインフラストラクチャ管理を可能にします
   - **追跡**：インフラストラクチャの変更を追跡できます
   - **バックアップ**：状態ファイルは S3 に安全にバックアップされます

## 現在の設定

このリポジトリの現在の構成はこれらのベストプラクティスに従っています：
- 状態ストレージに S3 を使用
- 状態ロックに DynamoDB を使用
- 機密情報は状態内のみに存在
- コードは構成とロジックのみを含む 