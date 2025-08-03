# EFS CSI Driver：手動 vs Terraform実装の比較

[English](../en/38_k8s_efs_csi_comparison.md) | [繁體中文](../zh-tw/38_k8s_efs_csi_comparison.md) | [日本語](../ja/38_k8s_efs_csi_comparison.md) | [インデックスに戻る](../README.md)

## 概要

この文書では、手動のEFS CSI driverインストール方法（[AWS EKS 実践ガイド](./35_eks_practice.md)より）と完全なTerraform実装を比較し、両者の違いを明確にし、なぜ手動アプローチが不完全なのかを説明します。

## 重要なポイント：手動アプローチは不完全

実装ガイドの手動インストールは、**CSI Driverの登録のみをインストールし、動的プロビジョニングを実行する実際のCSI Controller**は展開しません。

## 手動 vs Terraform実装の比較

### 手動アプローチ（実装ガイドより）

```bash
# ステップ1：EFSファイルシステムを手動作成（AWS Console）
# ステップ2：セキュリティグループを手動作成（AWS Console）  
# ステップ3：CSI Driverの登録のみをインストール
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/deploy/kubernetes/base/csidriver.yaml

# ステップ4：静的PVを手動作成
$ kubectl apply -f aws-efs-volume-pv.yaml

# ステップ5：PVCを手動作成
$ kubectl apply -f simple-volume-pvc.yaml
```

**実際にインストールされるもの：**
- ✅ CSI Driverの登録のみ
- ❌ CSI Controller（実際のプロビジョナー）なし
- ❌ 権限用のService Accountなし
- ❌ 動的プロビジョニング機能なし

### Terraform実装（完全なソリューション）

```hcl
# 自動化されたインフラストラクチャ作成
resource "aws_efs_file_system" "eks" { ... }           # EFSファイルシステム
resource "aws_security_group" "efs" { ... }            # セキュリティグループ
resource "aws_efs_mount_target" "eks" { ... }          # マウントターゲット
resource "aws_efs_access_point" "eks" { ... }          # アクセスポイント

# 完全なCSIインフラストラクチャ
resource "kubernetes_service_account" "efs_csi_controller" { ... }    # 権限管理
resource "kubernetes_deployment" "efs_csi_controller" { ... }         # Controller
resource "kubernetes_manifest" "efs_csi_driver" { ... }              # Driver登録
resource "kubernetes_storage_class" "efs" { ... }                    # 動的プロビジョニング
```

**提供されるもの：**
- ✅ 完全なEFSインフラストラクチャ
- ✅ 完全なCSI Controller展開
- ✅ 適切な権限を持つService Account
- ✅ 動的プロビジョニング機能
- ✅ アプリケーション用のStorage Class

## 詳細コンポーネント分析

### 1. EFSインフラストラクチャ (`efs.tf`)

#### `aws_efs_file_system`
```hcl
resource "aws_efs_file_system" "eks" {
  creation_token   = "${local.prefix}-efs"
  performance_mode = "generalPurpose"
  throughput_mode  = "elastic"           # 自動スケーリングスループット
  encrypted        = true                # 保存時暗号化
}
```
**目的**：EKSワークロードに最適な設定で実際のEFSファイルシステムを作成。

#### `aws_security_group`
```hcl
resource "aws_security_group" "efs" {
  ingress {
    from_port   = 2049                   # NFSポート
    to_port     = 2049
    protocol    = "tcp"
    cidr_blocks = [data.terraform_remote_state.infra_networking.outputs.vpc_cidr]
  }
}
```
**目的**：EFSへのネットワークアクセスを制御し、EKSノードからのNFSトラフィックのみを許可。

#### `aws_efs_mount_target`
```hcl
resource "aws_efs_mount_target" "eks" {
  count           = length(local.vpc_config_from_remote.private_subnet_ids)
  file_system_id  = aws_efs_file_system.eks.id
  subnet_id       = local.vpc_config_from_remote.private_subnet_ids[count.index]
  security_groups = [aws_security_group.efs.id]
}
```
**目的**：各アベイラビリティゾーンにマウントターゲットを作成し、高可用性を確保。

#### `aws_efs_access_point`
```hcl
resource "aws_efs_access_point" "eks" {
  file_system_id = aws_efs_file_system.eks.id
  posix_user {
    gid = 1000
    uid = 1000
  }
  root_directory {
    path = "/app"
    creation_info {
      owner_gid   = 1000
      owner_uid   = 1000
      permissions = "0755"
    }
  }
}
```
**目的**：EFSファイルシステムへの安全で制御されたエントリポイントを特定の権限で提供。

### 2. CSI Controllerインフラストラクチャ (`efs-csi.tf`)

#### `kubernetes_service_account`
```hcl
resource "kubernetes_service_account" "efs_csi_controller" {
  metadata {
    name      = "${local.prefix}-efs-csi-controller-sa"
    namespace = "kube-system"
    labels = {
      "app.kubernetes.io/name" = "aws-efs-csi-driver"
    }
  }
}
```
**目的**：CSI controllerにボリューム管理に必要なKubernetes権限を提供。

#### `kubernetes_deployment` (CSI Controller)
```hcl
resource "kubernetes_deployment" "efs_csi_controller" {
  spec {
    replicas = 2 # 高可用性
    template {
      spec {
        service_account_name = kubernetes_service_account.efs_csi_controller.metadata[0].name
        
        container {
          name  = "${local.prefix}-efs-plugin"
          image = "amazon/aws-efs-csi-driver:v2.0.7"
          # ボリューム操作を処理するCSI controller
        }
        
        container {
          name  = "${local.prefix}-csi-provisioner"
          image = "registry.k8s.io/sig-storage/csi-provisioner:v4.0.1"
          # Kubernetes CSI provisionerサイドカー
        }
      }
    }
  }
}
```
**目的**：
- **EFS Plugin**：AWS EFS固有の操作を処理
- **CSI Provisioner**：PVC要求を監視し、ボリューム作成をトリガーする標準的なKubernetes CSIサイドカー

#### `kubernetes_manifest` (CSI Driver)
```hcl
resource "kubernetes_manifest" "efs_csi_driver" {
  manifest = {
    apiVersion = "storage.k8s.io/v1"
    kind       = "CSIDriver"
    metadata = {
      name = "efs.csi.aws.com"
    }
    spec = {
      attachRequired = false             # EFSはアタッチメントが不要
    }
  }
}
```
**目的**：EFS CSI driverをKubernetesに登録し、EFSボリュームの処理方法を指示。

#### `kubernetes_storage_class`
```hcl
resource "kubernetes_storage_class" "efs" {
  metadata {
    name = "${local.prefix}-efs-sc"
  }
  storage_provisioner = "efs.csi.aws.com"
  parameters = {
    provisioningMode = "efs-ap"          # アクセスポイントを使用
    fileSystemId     = aws_efs_file_system.eks.id
    directoryPerms   = "700"
  }
}
```
**目的**：動的PV作成のテンプレートを提供。アプリケーションがこのstorage classを参照するPVCを作成すると、Kubernetesは対応するPVを自動的に作成。

## なぜ手動アプローチは単純すぎるのか？

### 不足している要素

1. **CSI Controllerなし**：手動アプローチはCSI Driverの登録のみをインストールし、プロビジョニング要求を処理する実際のcontrollerがない。

2. **Service Accountなし**：適切なservice accountがないため、CSIコンポーネントに必要な権限がない。

3. **動的プロビジョニングなし**：ユーザーは静的PVを手動作成し、EFSファイルシステムIDをハードコードする必要がある。

4. **Storage Classなし**：アプリケーションが動的にストレージを要求できない。

### 適切な手動インストールは以下であるべき

- 以下のファイル内容のとおり：
https://github.com/kubernetes-sigs/aws-efs-csi-driver/blob/master/deploy/kubernetes/overlays/stable/kustomization.yaml

- またはHelmを使用：
  ```bash
  $ helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
  $ helm install aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver
  ```

## アプリケーション使用方法の比較

### 手動アプローチ（静的プロビジョニング）
```yaml
# PVを手動作成
apiVersion: v1
kind: PersistentVolume
metadata:
  name: app-pv
spec:
  storageClassName: sc-001
  capacity:
    storage: 2Gi
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-0a1700a34bf9e8d24  # ハードコードされたEFS ID
---
# その後PVCを作成
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc
spec:
  storageClassName: sc-001
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
```

### Terraformアプローチ（動的プロビジョニング）
```yaml
# アプリケーションはPVCのみが必要 - PVは自動的に作成される
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: prod-app-pvc
  namespace: app-ns
spec:
  storageClassName: dev-eks-storage-efs-sc  # Terraform作成のstorage classを参照
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
```

## Terraform実装の利点

### 1. **完全なInfrastructure as Code**
- すべてのリソースが定義され、バージョン管理される
- 環境間で再現可能
- 自動依存関係管理

### 2. **動的プロビジョニング**
- アプリケーションがストレージ要件を自己記述
- 手動でのPV作成が不要
- PVC削除時の自動クリーンアップ

### 3. **適切なセキュリティ**
- 最小限の必要なアクセス権を持つセキュリティグループ
- 最小権限の原則に従うservice account
- デフォルトでの暗号化ストレージ

### 4. **高可用性**
- 複数のアベイラビリティゾーンにわたるマウントターゲット
- 冗長なCSI controllerレプリカ
- 弾性スループットスケーリング

### 5. **本番運用対応**
- 完全なログ記録と監視
- 適切なリソース制限と要求
- ライフサイクル管理ポリシー

## 結論

実装ガイドの手動アプローチは簡単な練習向けであり、**不完全で本番環境には適していません**。実際のプロビジョニングインフラストラクチャなしにCSI Driverの登録のみをインストールし、ユーザーに静的PVの手動作成を強制します。

Terraform実装は以下をサポートする**完全で本番運用対応のEFS CSIソリューション**を提供します：
- ✅ 完全な自動化
- ✅ 動的プロビジョニング
- ✅ 高可用性
- ✅ セキュリティベストプラクティス
- ✅ Infrastructure as Code原則

**重要なポイント**：手動インストールチュートリアルが基本的な登録ステップだけでなく、すべての必要なコンポーネントを含んでいることを常に確認してください。