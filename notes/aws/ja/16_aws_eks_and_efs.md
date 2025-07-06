# AWS EKS と EFS 永続ストレージ統合

[English](../en/16_aws_eks_and_efs.md) | [繁體中文](../zh-tw/16_aws_eks_and_efs.md) | [日本語](../ja/16_aws_eks_and_efs.md) | [インデックスに戻る](../README.md)

## 概要

Kubernetes環境では、Podのライフサイクルは一時的で、Podが再起動または移行するとコンテナ内のデータが失われます。データの永続的な保存を実現するため、AWS EKSはEFS（Elastic File System）との統合ソリューションを提供し、CSI（Container Storage Interface）ドライバーを通じてストレージリソースを管理します。

## アーキテクチャ図

```                                                          
              Admin EC2                                   Cluster                                                                     
  +-------------------------------+    +------------------------------------------------+                                             
  | +--------+                    |    |                                                |                                             
  | | eksctl |                    |    |                                 PV             |                                             
  | +--------+                    |    |                             +--------+     +-------+      +----------+      +-------------+  
  |  |                            |    |         Pod                 | static |---->|  csi  |----->| Security |----->| AWS EFS     |  
  |  |--create k8s cluster -------|--->|  +---------------+          +--------+     +-------+      | Group    |      | File System |  
  |  |                            |    |  | +-----------+ |               |             |          +----------+      +-------------+  
  |  +--create cluster resources -|--->|  | | container | |               |             |                                             
  |                               |    |  | +-----------+ |       +--------------+      |                                             
  | +---------+                   |    |  |   |           |       | StorageClass |      |                                             
  | | kubectl |                   |    |  |   |-image     |       +--------------+      |                                             
  | +---------+                   |    |  |   |           |               |             |                                             
  |  |                            |    |  |   +-Volume ---|-----\         |             |                                             
  |  +--access k8s cluster -------|--->|  +---------------+      ---\  +-----+          |                                             
  |                               |    |                             ->| PVC |          |                                             
  +-------------------------------+    |                               +-----+          |                                             
                                       |                                                |                                             
                                       |                                                |                                             
                                       +------------------------------------------------+                                                                
```

## コアコンポーネント説明

#### CSI（Container Storage Interface）ドライバー

CSIはKubernetesとストレージシステム間の標準化されたインターフェースです。AWS EFS CSI DriverはEKS専用に設計されたドライバーで、以下の責任を負います：

- **ストレージリソース管理**：EFSファイルシステムの動的作成と管理
- **マウント操作**：EFSファイルシステムをPodにマウント
- **アクセス制御**：EFSのアクセス権限とセキュリティグループの管理
- **ライフサイクル管理**：ストレージリソースの作成、削除、更新の処理

#### StorageClass

StorageClassは動的ストレージプロビジョニングのテンプレートを定義します：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-xxxxx
  directoryPerms: "700"
```

**重要なパラメータ説明：**
- `provisioner`: EFS CSIドライバーの使用を指定
- `provisioningMode`: プロビジョニングモード（efs-apはAccess Pointの使用を意味）
- `fileSystemId`: EFSファイルシステムID
- `directoryPerms`: ディレクトリ権限設定

#### PersistentVolumeClaim（PVC）

PVCはPodがストレージリソースを要求する宣言です：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```

**主要な特徴：**
- `ReadWriteMany`: 複数のPodが同時に読み書きをサポート
- `storageClassName`: 使用するStorageClassを指定
- `storage`: 要求するストレージ容量（EFSは実際には使用量ベースで課金）

#### PersistentVolume（PV）

PVは実際のストレージリソースで、CSIドライバーによって動的に作成されます：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-xxxxx::fsap-xxxxx
```

## CSIドライバーの利点

#### 標準化されたインターフェース
- Kubernetes CSI標準に準拠
- 他のストレージシステムとの互換性
- ストレージ管理プロセスの簡素化

#### 動的プロビジョニング
- オンデマンドでストレージリソースを作成
- PVライフサイクルの自動管理
- 手動設定作業の削減

#### マルチテナンシーサポート
- Access Pointによる分離の実現
- 異なる権限設定のサポート
- セキュリティの向上

#### 高可用性
- アベイラビリティーゾーン間のデータレプリケーション
- 自動フェイルオーバー
- 99.99%の可用性保証

## セキュリティ考慮事項

#### ネットワークセキュリティ
- VPCエンドポイントを使用してネットワーク遅延を削減
- 適切なセキュリティグループルールの設定
- EFSのネットワークアクセス制限

#### アクセス制御
- IAMロールによる認証の使用
- EFS Access Point権限の設定
- 最小権限原則の実装

#### データ暗号化
- 転送中暗号化（TLS）の有効化
- 静止中暗号化（KMS）の有効化
- 暗号化キーの定期的なローテーション

## ベストプラクティス

#### パフォーマンス最適化
- EFSパフォーマンスモード（General PurposeまたはMax I/O）の使用
- 適切なスループットモードの設定
- I/Oパフォーマンス指標の監視

#### コスト管理
- 適切なストレージクラスの選択
- ストレージ使用量の監視
- ライフサイクルポリシーの実装

#### バックアップ戦略
- 定期的なEFSスナップショットの作成
- 災害復旧手順のテスト
- データ保持ポリシーの確立

## トラブルシューティング

#### 一般的な問題

1. **PVCがバインドできない**
- StorageClass設定の確認
- EFSファイルシステム状態の検証
- CSIドライバーが正常に動作しているかの確認

2. **PodがEFSをマウントできない**
- セキュリティグループルールの確認
- ネットワーク接続の検証
- IAM権限設定の確認

3. **パフォーマンス問題**
- EFSパフォーマンス指標の監視
- アプリケーションI/Oパターンの調整
- EFSパフォーマンスモードの使用を検討

## まとめ

AWS EFS CSI Driverを通じて、EKSクラスターは簡単に永続的なデータストレージを実現できます。CSIドライバーは標準化されたインターフェースを提供し、ストレージ管理プロセスを簡素化しながら、データの信頼性とセキュリティを確保します。EFSの適切な設定と使用により、コンテナ化されたアプリケーションに高性能で高可用性のストレージソリューションを提供できます。