# Terraform Provisionerと条件ロジックの説明

[English](../en/04_about_provisioner_local-exec.md) | [繁體中文](../zh-tw/04_about_provisioner_local-exec.md) | [日本語](../ja/04_about_provisioner_local-exec.md) | [インデックスに戻る](../README.md)

## 概要

本文書では、Terraformにおける`provisioner "local-exec"`の使用方法と、Terraformが`count`と三項演算子を通じて条件ロジックを実装する方法について説明します。

## Provisioner "local-exec" 詳細解説

### 基本概念

`provisioner "local-exec"`はTerraformの組み込みプロビジョナーで、リソースのライフサイクルの特定のタイミングでローカルシェルコマンドを実行するために使用されます。

### 構文構造

```hcl
provisioner "local-exec" {
  command = "実行するコマンド"
  # その他のオプションパラメータ
}
```

### 実行タイミング

- **create**: リソース作成後に実行（デフォルト）
- **destroy**: リソース削除前に実行
- **update**: リソース更新後に実行

### 予約語の説明

`local-exec`はTerraformの**予約語**であり、カスタム用語ではありません。Terraformの組み込みプロビジョナータイプの一つです。

## 条件ロジックの実装方法

### countを使用した条件作成

```hcl
resource "null_resource" "vpc_config_validation" {
  count = local.vpc_config_valid ? 0 : 1

  provisioner "local-exec" {
    command = "echo 'Error: Invalid VPC configuration. Please ensure networking module is deployed and vpc_config is properly configured.' && exit 1"
  }
}
```

### ロジック分析

| `local.vpc_config_valid` | `count` | 結果 |
|--------------------------|---------|------|
| `true` | `0` | **リソース未作成** → **プロビジョナー未実行** |
| `false` | `1` | **リソース作成** → **プロビジョナー実行** |

### 三項演算子構文

```hcl
count = local.vpc_config_valid ? 0 : 1
```

これは以下と同等です：
```hcl
if local.vpc_config_valid == true:
  count = 0  # リソースを作成しない
else:
  count = 1  # 1つのリソースを作成
```

## Terraform条件ロジック設計思想

### なぜ従来のif-elseを使用しないのか？

1. **宣言的言語の特性**
   - Terraformは命令的（imperative）ではなく宣言的（declarative）言語
   - 「実行過程」ではなく「目標状態」の記述に焦点を当てる

2. **状態管理の優先**
   - インフラストラクチャの「状態」管理に焦点を当てる
   - リソース状態の一貫性と予測可能性を確保

3. **冪等性の保証**
   - 何度実行しても同じ結果
   - 命令的構文が引き起こす可能性のある副作用を回避

## 一般的な条件ロジックパターン

### 1. 三項演算子（最も一般的）

```hcl
# if-elseの代替
count = var.environment == "prod" ? 1 : 0

# if-elseif-elseの代替
name = var.environment == "prod" ? "prod-cluster" : 
       var.environment == "staging" ? "staging-cluster" : 
       "dev-cluster"
```

### 2. count条件作成

```hcl
# if condition: create resourceの代替
resource "aws_instance" "conditional" {
  count = var.create_instance ? 1 : 0
  # リソース設定...
}
```

### 3. for_each動的作成

```hcl
# for loopの代替
resource "aws_instance" "multiple" {
  for_each = var.instance_configs
  # リソース設定...
}
```

### 4. dynamicブロック

```hcl
# ネストしたif-elseの代替
resource "aws_instance" "example" {
  dynamic "ebs_block_device" {
    for_each = var.create_ebs ? [1] : []
    content {
      # ブロック内容...
    }
  }
}
```

## 実際の応用例

### エラーハンドリングメカニズム

```hcl
# VPC設定の検証
resource "null_resource" "vpc_config_validation" {
  count = local.vpc_config_valid ? 0 : 1

  provisioner "local-exec" {
    command = "echo 'Error: Invalid VPC configuration. Please ensure networking module is deployed and vpc_config is properly configured.' && exit 1"
  }
}
```

### 環境固有の設定

```hcl
resource "aws_eks_cluster" "main" {
  count = var.environment == "prod" ? 1 : 
          var.environment == "staging" ? 1 : 1
  
  name = var.environment == "prod" ? "prod-cluster" :
         var.environment == "staging" ? "staging-cluster" :
         "dev-cluster"
}
```

## ベストプラクティス

1. **三項演算子を使用**してシンプルな条件判断を行う
2. **countを使用**して条件的リソース作成を行う
3. **for_eachを使用**して動的リソース作成を行う
4. **dynamicブロックを使用**して複雑な条件ロジックを処理する
5. **コードの可読性を保持**し、複雑な条件ロジックには適切なコメントを追加する

## まとめ

Terraformの条件ロジック設計は、その宣言的言語の本質を反映しています：
- 「どうするか」ではなく「何が欲しいか」を記述することに焦点を当てる
- 条件演算子と動的ブロックを通じて複雑なロジックを実装する
- インフラストラクチャ設定の予測可能性と一貫性を確保する

この設計により、Terraformは複雑なインフラストラクチャ設定の管理により適しており、同時にコードの可読性と保守性を維持します。