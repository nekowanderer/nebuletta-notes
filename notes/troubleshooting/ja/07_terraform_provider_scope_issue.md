# Terraform Providerスコープによる「no client config」エラー

[English](../en/07_terraform_provider_scope_issue.md) | [繁體中文](../zh-tw/07_terraform_provider_scope_issue.md) | [日本語](../ja/07_terraform_provider_scope_issue.md) | [インデックスに戻る](../README.md)

---

## 背景
- 実験日：2025/08/03
- 難易度：🤬
- 説明：Kubernetes provider設定を共有モジュールとして抽出しようとした際に「cannot create REST client: no client config」エラーが発生。

---

## 問題の説明

Terraformアーキテクチャの設計において、複数のスタックでKubernetes provider設定の重複を避けるため、共有providerモジュールの作成を試みたが、Kubernetesリソースが正しく初期化されない問題に遭遇。

### エラーメッセージ例

```
╷
│ Error: Failed to construct REST client
│
│   with module.eks_storage.kubernetes_manifest.efs_csi_driver,
│   on ../../../../modules/eks-lab/storage/efs-csi.tf line 160, in resource "kubernetes_manifest" "efs_csi_driver":
│  160: resource "kubernetes_manifest" "efs_csi_driver" {
│
│ cannot create REST client: no client config
╵
```

### 誤ったアーキテクチャ設計

```hcl
# ❌ 誤り：provider設定を子モジュールに配置しようとしている
module "eks_providers" {
  source = "../../../../modules/shared/eks-providers"
  
  cluster_name           = data.terraform_remote_state.eks_cluster.outputs.cluster_name
  cluster_endpoint       = data.terraform_remote_state.eks_cluster.outputs.cluster_endpoint
  cluster_ca_certificate = data.terraform_remote_state.eks_cluster.outputs.cluster_certificate_authority_data
  aws_region            = "ap-northeast-1"
}

module "eks_storage" {
  source = "../../../../modules/eks-lab/storage"
  # ❌ このモジュールは上記の子モジュールのprovider設定を使用できない
  depends_on = [module.eks_providers]
}
```

---

## 根本原因

### 1. Terraform Providerスコープルール

TerraformのProvider設定には厳格なスコープ制限があります：

- **モジュール内部分離**：各モジュールは独自のproviderスコープを持つ
- **下方継承**：親レベルのprovider設定は子モジュールに自動的に渡される
- **横方向共有不可**：子モジュールや並列モジュールのprovider設定は互いに影響しない

### 2. `depends_on`機能の誤解

```
❌ 誤った認識：depends_on = [module.eks_providers] で設定共有が可能
✅ 実際の状況：depends_onは実行順序のみを制御し、providerスコープには影響しない
```

### 3. Provider初期化フロー

```
1. Terraformがkubernetes_manifestリソースを読み取り
2. 現在のスコープでkubernetes provider設定を検索
3. 設定が見つからない → "cannot create REST client: no client config"
4. depends_on依存関係があっても、他のモジュールのproviderにはアクセスできない
```

---

## 解決手順

### 解決策1：スタックレベルでの直接Provider設定

```hcl
# ✅ 正解：スタックレベルでproviderを直接設定
provider "kubernetes" {
  host                   = data.terraform_remote_state.eks_cluster.outputs.cluster_endpoint
  cluster_ca_certificate = base64decode(data.terraform_remote_state.eks_cluster.outputs.cluster_certificate_authority_data)
  
  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "aws"
    args = [
      "eks",
      "get-token",
      "--cluster-name",
      data.terraform_remote_state.eks_cluster.outputs.cluster_name,
      "--region",
      "ap-northeast-1"
    ]
  }
}

module "eks_storage" {
  source = "../../../../modules/eks-lab/storage"
  # ✅ モジュールは同レベルのprovider設定を自動継承
}
```

### 解決策2：Terramate共有テンプレートの使用

#### ステップ1：共有テンプレートの作成

ファイル `stacks/dev/shared/terramate-templates/eks-k8s-provider.tm.hcl` を作成：

```hcl
generate_hcl "_terramate_generated_k8s_provider.tf" {
  content {
    provider "kubernetes" {
      host                   = data.terraform_remote_state.eks_cluster.outputs.cluster_endpoint
      cluster_ca_certificate = base64decode(data.terraform_remote_state.eks_cluster.outputs.cluster_certificate_authority_data)

      exec {
        api_version = "client.authentication.k8s.io/v1beta1"
        command     = "aws"
        args = [
          "eks",
          "get-token",
          "--cluster-name",
          data.terraform_remote_state.eks_cluster.outputs.cluster_name,
          "--region",
          global.aws_region
        ]
      }
    }
  }
}
```

#### ステップ2：必要なスタックでのインポート

`stack.tm.hcl` に追加：

```hcl
# 共有providerテンプレートのインポート
import {
  source = "../../shared/terramate-templates/eks-k8s-provider.tm.hcl"
}
```

#### ステップ3：Terramate生成の実行

```bash
$ terramate generate
```

---

## 技術的原理

### Providerスコープルール図

```
┌──────────────────────────────────────────────────┐
│                スタックレベル                      │
│  ┌─────────────────────────────────────────────┐ │
│  │  provider "kubernetes" { ... }              │ │
│  └─────────────────────┬───────────────────────┘ │
│                        │ (下方継承)               │
│  ┌─────────────────────▼───────────────────────┐ │
│  │           module "business_logic"           │ │
│  │  ✅ 上位のprovider設定を使用可能               │ │
│  └─────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│                スタックレベル                      │
│  ┌─────────────────────┐ ┌─────────────────────┐ │
│  │  module "providers" │ │  module "business"  │ │
│  │  内部にprovider有り   │ │  ❌ 左側にアクセス    │ │
│  └─────────────────────┘ └─────────────────────┘ │
│              │                       ▲           │
│              └───── depends_on ──────┘           │
│                   (実行順序のみ制御)                │
└──────────────────────────────────────────────────┘
```

### Kubernetes Provider初期化フロー

```
1. Terraformがrequired_providersをスキャン
   ↓
2. 対応するprovider "kubernetes"設定を検索
   ↓
3. host、execパラメータを使用してHTTPクライアントを作成
   ↓ 
4. aws eks get-tokenを呼び出してJWTトークンを取得
   ↓
5. トークンをAuthorizationヘッダーに追加
   ↓
6. EKS APIサーバーにリクエストを送信
```

ステップ2でprovider設定が見つからない場合、「no client config」エラーが発生します。

---

## 予防措置

### 1. Providerスコープの理解

- ✅ **正解**：使用する層でproviderを設定
- ❌ **誤り**：子モジュールやdepends_onを通じたprovider共有を期待

### 2. 適切な共有戦略の選択

| 解決策 | 利点 | 欠点 | 適用場面 |
|--------|------|------|----------|
| スタックレベル設定 | シンプルで直接的 | 重複の可能性 | 単一プロジェクト |
| Terramateテンプレート | 重複回避、統一管理 | ツールサポートが必要 | マルチ環境プロジェクト |
| グローバルprovider | 完全共有 | 柔軟性不足 | 標準化環境 |

### 3. 設計原則

- **Provider近接の原則**：使用場所に最も近い層で設定
- **モジュール間依存の回避**：モジュール間でのprovider共有を期待しない
- **設定のドキュメント化**：各providerの設定源を明確に記録

---

## 関連概念

### Terraformモジュールシステム

```bash
# モジュール依存関係ツリーの表示
$ terraform graph

# provider設定の確認
$ terraform providers

# 設定正確性の検証
$ terraform validate
```

### Provider設定ベストプラクティス

```hcl
# ✅ 推奨：明示的なprovider設定
terraform {
  required_providers {
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.20"
    }
  }
}

provider "kubernetes" {
  # 明示的な接続情報
}

# ❌ 回避：暗黙的継承やモジュール間共有への依存
```

---

## トラブルシューティング

### provider設定問題が依然として発生する場合：

1. **スコープの確認**：provider設定と使用が同レベルにあることを確認
2. **構文の検証**：`terraform validate`で設定正確性を確認
3. **権限の確認**：AWS認証とEKSアクセス権限が正常であることを確認
4. **キャッシュのクリア**：`.terraform`ディレクトリを削除して再初期化
5. **段階的デバッグ**：シンプルなprovider設定から始めて、複雑なロジックを追加

### デバッグコマンド

```bash
$ terraform init -upgrade    # 再初期化とproviderアップグレード
$ terraform providers        # provider設定状態の表示
$ terraform plan -var-file=... # 詳細な実行プランの表示
$ TF_LOG=DEBUG terraform plan   # 詳細ログの有効化
```