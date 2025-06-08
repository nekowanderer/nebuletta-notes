# Terraform 基本コマンド

[English](../en/01_basic_terraform_commands.md) | [繁體中文](../zh-tw/01_basic_terraform_commands.md) | [日本語](01_basic_terraform_commands.md) | [索引に戻る](../README.md)

### コマンド
#### init
初期化コマンドはすべての Terraform モジュールに必要です：
```bash 
$ terraform init 
```
初期化フェーズでは、Terraform はプロバイダーをダウンロードします。一般的に、Terraform コアは抽象化にすぎません；リソースの具体的な動作とその相互作用はプロバイダーによって決定されます。

#### plan
Plan コマンドはすべての .tf ファイルとパブリッククラウド上の実際の内容を比較しようとします。違いを表示し、変更に関する計画を出力します。
```bash
$ terraform plan
```

#### apply
計画が期待通りであれば、つまりすべての .tf ファイルが適切であれば、必要なリソースを生成するためにパブリッククラウドに対して apply コマンドを実行できます。「yes」と入力する前に、すべての適用情報を確認してください。
```bash
$ terraform apply
```

#### destroy
```bash
$ terraform destroy
```
Destroy コマンドは、現在の Terraform 構成で管理されているすべてのリソースを削除します。本質的には、すべてを削除することを目標とするプランを実行しています。Terraform は削除計画を表示し、apply コマンドと同様に続行前に確認を求めます。特に本番環境では注意して使用してください。

#### 一般的なプロセス
`init` -> `plan` -> `apply` -> `destroy`（オプション） 