# Terraform状態の誤設定によるリソース削除の問題

[English](../en/03_incorrect_shared_state_behavior.md) | [繁體中文](../zh-tw/03_incorrect_shared_state_behavior.md) | [日本語](03_incorrect_shared_state_behavior.md) | [インデックスに戻る](../README.md)

---

## 背景

- 日付：2025/05/24
- 難易度：🤬🤬
- 説明：Terraformで複数のモジュール（`infra-networking` と `isolated-ec2`）を管理する際、バックエンドの設定が不適切だったため、状態ファイルが重複して管理され、意図しないリソースの削除が発生しました。

---

## 発生した問題

- `infra-networking` モジュールを正常に apply した後、VPC やサブネット、ルートテーブルなどのリソースが AWS Console にて確認できた。
- 続いて `isolated-ec2` モジュールで `terraform plan` を実行すると、Terraform が `infra-networking` により作成されたリソースを全て削除しようとした。
- これは我々が意図した「積み上げ型」の設計ではなく、2つのモジュールが同じコンテキストを共有していたことが原因。

---

## 誤設定の構成図

### ❌ infra-network と isolated-ec2 が同じ状態ファイルを共有している場合

📁 state-storage/
└── dev-backend.hcl     --> 共通バックエンド設定

📁 infra-network/
└── main.tf             --> dev-backend.hcl を使用

📁 isolated-ec2/
└── main.tf             --> 同様に dev-backend.hcl を使用

📦 AWS S3
└── key = dev/terraform.tfstate
    └── infra と ec2 の両方のリソースを含む

```
          +-----------------------------+
          |  S3: dev/terraform.tfstate  |
          +-----------------------------+
                  ▲          ▲
                  |          |
       +----------+          +----------+
       |                                |
+---------------+             +------------------+
| infra-network |             |  isolated-ec2    |
+---------------+             +------------------+
         （状態ファイルを共有し、相互に影響）
```

---

## 正しい構成

### ✅ infra-network と isolated-ec2 に個別の状態ファイルを割り当てる

📁 state-storage/
├── dev-infra.hcl       --> infra-network 用
└── dev-ec2.hcl         --> isolated-ec2 用

📁 infra-network/
└── main.tf             --> dev-infra.hcl を使用

📁 isolated-ec2/
└── main.tf             --> dev-ec2.hcl を使用

📦 AWS S3
├── key = dev/infra/terraform.tfstate
└── key = dev/ec2/terraform.tfstate

          +---------------------------------+       +--------------------------------+
          | S3: dev/infra/terraform.tfstate |       | S3: dev/ec2/terraform.tfstate  |
          +---------------------------------+       +--------------------------------+
                           ▲                                        ▲
                           |                                        |
                   +---------------+                       +------------------+
                   | infra-network |                       |  isolated-ec2    |
                   +---------------+                       +------------------+
                                          （論理的に分離）

---

## デバッグの流れ

- 初めに `infra-networking` のみがバックエンドを設定しており、S3 に状態が保存された。
- `isolated-ec2` 側は独自の backend.hcl を持たず、デフォルト（または `dev-backend.hcl`）を使用。
- Terraform は全ての状態を `isolated-ec2` のものと解釈し、既存リソースを「同期」する形で削除しようとした。

---

## 解決方法

1. `isolated-ec2` 用に独立した backend 設定ファイル（HCL）を作成。
2. 環境・モジュールごとに backend を自動生成するスクリプトを導入。
3. 各 backend 設定にユニークな key と DynamoDB Lock Table を割り当てて干渉を防止。
4. 修正後に `terraform init` → `plan` を実行すると、リソースの削除は発生しなくなった。

---

## 補足

この構成では、backend 設定は「環境 + モジュール」単位で管理されており、シェルスクリプトにより自動生成されています。

このスクリプトの詳細は `terraform/envs` フォルダにて別途記載予定 ([README](../../../envs/README.md))。

---

## まとめ

Terraform で複数のモジュールが同一の backend を共有すると、適切な分離がなされていない限り、リソース削除などの意図しない挙動が発生するリスクがある。

**教訓**：モジュール単位での状態管理・分離は、信頼性の高いインフラ構成の鍵となる。
