# AWS ECS Cluster Capacity Provider

[English](../en/20_aws_ecs_cluster_capacity_provider.md) | [繁體中文](../zh-tw/20_aws_ecs_cluster_capacity_provider.md) | [日本語](../ja/20_aws_ecs_cluster_capacity_provider.md) | [インデックスに戻る](../README.md)

## Capacity Providerとは？

AWS ECS Capacity Providerは、ECS Cluster内のコンピューティングリソースの配置とスケーリングを制御する**自動化されたリソース管理メカニズム**です。**コンテナがどのインフラストラクチャ上で実行されるか**、および**リソースの規模を自動的に調整する方法**を決定します。

---

## 主なタイプ

### 1. EC2 Capacity Provider
- **EC2インスタンスベース**
- 完全なオペレーティングシステム制御権を提供
- 特定の設定や永続化ニーズが必要なアプリケーションに適用

### 2. Fargate Capacity Provider
- **サーバーレスコンテナサービス**
- AWSがインフラストラクチャを完全管理
- 従量課金制、EC2インスタンスの管理不要

### 3. Fargate Spot Capacity Provider
- **Fargate Spotインスタンスベース**
- コストが低いが中断される可能性あり
- 耐障害性の高いワークロードに適用

---

## 核心機能

### 自動スケーリング（Auto Scaling）
```
目標容量 ──→ [Capacity Provider] ──→ 実際のリソース配置
     ↑                                    ↓
   監視メトリクス ←─────── [CloudWatch] ←─────── リソース使用状況
```

### リソース配置戦略

#### REPLICA戦略
- **すべてのCapacity Providerにタスクを均等分散**
- 高可用性要件に適し、リスクを分散

#### BINPACK戦略
- **既存リソースを優先的に満杯にしてから新しいリソースを開始**
- スーツケースの荷造りのように、既存のスペースを先に満たす

### BINPACK詳細説明

#### 動作原理
以下のECS Cluster設定があると仮定：
```
EC2インスタンス仕様：4 vCPU, 8GB RAM

現在の状況：
Instance-1: 使用済み 2 vCPU, 4GB RAM (残り 2 vCPU, 4GB)
Instance-2: 使用済み 1 vCPU, 2GB RAM (残り 3 vCPU, 6GB)
```

新しいタスクが`1 vCPU, 2GB RAM`を必要とする場合：

**BINPACKが選択**：
- Instance-1を選択（使用率の高いマシン）
- 結果：Instance-1の使用率が75%に向上

**REPLICAが選択**：
- Instance-2を選択（負荷を平均分散）
- 結果：両方のマシンの使用率が中程度を維持

#### 実際のケース比較
```
シナリオ：10個のタスクをデプロイ

BINPACK結果：
├── Instance-1: 100% 使用率 (8個のタスク) ✓ フル稼働
├── Instance-2: 100% 使用率 (2個のタスク) ✓ フル稼働  
└── Instance-3: 0% 使用率 → シャットダウンしてコスト節約

REPLICA結果：
├── Instance-1: 33% 使用率 (3-4個のタスク)
├── Instance-2: 33% 使用率 (3-4個のタスク)
└── Instance-3: 33% 使用率 (2-3個のタスク) → すべて稼働維持が必要
```

#### BINPACKの利点
- **コスト節約**：アイドルマシンをシャットダウン可能
- **効率向上**：リソース統合度が高い
- **弾力的ワークロードに適用**：Auto Scalingと組み合わせて使用
- **Spotインスタンス対応**：中断影響範囲を削減

#### BINPACK注意事項
- **単一障害点リスク**：ホットスポットマシンの障害影響が大きい
- **リソース競合**：パフォーマンスボトルネックを引き起こす可能性
- **重要サービスには不適**：高可用性が必要なアプリケーションは慎重に使用

#### 使用推奨事項
- **適用シナリオ**：開発テスト環境、コスト敏感アプリケーション、バッチ処理ジョブ
- **不適用シナリオ**：本番重要サービス、高可用性が必要なアプリケーション

---

## 設定要素

### 基本設定
| 項目 | 説明 |
|------|------|
| **Provider Name** | Capacity Provider識別名 |
| **Auto Scaling Group** | 関連するEC2 Auto Scaling Group（EC2タイプのみ） |
| **Managed Scaling** | 自動スケーリングを有効にするか |
| **Target Capacity** | 目標容量パーセンテージ（通常100%に設定） |

### 高度な設定
```bash
# 例：EC2 Capacity Providerの作成
$ aws ecs create-capacity-provider \
    --name my-ec2-capacity-provider \
    --auto-scaling-group-provider \
        autoScalingGroupArn=arn:aws:autoscaling:region:account:autoScalingGroup \
        managedScaling=enabled,status=ENABLED,targetCapacity=100 \
        managedTerminationProtection=ENABLED
```

---

## 実際の動作フロー

### 1. タスクリクエスト段階
```
ECS Service/Task → Cluster → Capacity Provider → リソース配置
```

### 2. リソース不足時
```
需要増加 → 容量不足検出 → スケーリングトリガー → コンピューティングリソース追加 → コンテナデプロイ
```

### 3. 需要減少時
```
負荷減少 → 過剰容量検出 → スケールダウントリガー → コンピューティングリソース回収 → コスト節約
```

---

## ハイブリッド戦略設定

### マルチProvider組み合わせ
```yaml
# 例：ハイブリッド使用戦略
Cluster:
  CapacityProviders:
    - EC2-Provider:      weight: 2
    - Fargate-Provider:  weight: 1
    - FargateSpot-Provider: weight: 1
```

### 使用シナリオ推奨
- **本番環境安定サービス**：EC2 + Fargate
- **開発テスト環境**：Fargate Spot
- **混合ワークロード**：3種類のProvider組み合わせ使用

---

## コスト最適化戦略

### 1. Spotインスタンス統合
- Fargate Spotを利用してコストを最大70%削減
- 適切な耐障害性メカニズムの設定

### 2. インテリジェントリソース配置
- アプリケーション特性に基づいて適切なProviderを選択
- 合理的なスケーリング戦略パラメータの設定

### 3. 監視と調整
```bash
# Capacity Provider使用状況の監視
$ aws ecs describe-capacity-providers \
    --capacity-providers my-capacity-provider
```

---

## 注意事項

### 設定制限
- EC2 Capacity Providerは有効なAuto Scaling Groupと関連付けが必要
- Managed Scaling有効化後、手動でのASG調整は非推奨

### ベストプラクティス
- 適切な`targetCapacity`値の設定（推奨100%）
- `managedTerminationProtection`を有効にして意図しない終了を回避
- CloudWatchメトリクスを定期的にチェックして調整

### トラブルシューティング
```bash
# Capacity Providerステータスの確認
$ aws ecs describe-clusters --cluster my-cluster \
    --include CAPACITY_PROVIDERS
```

---

## まとめ

Capacity ProviderはECS自動化運用の核心メカニズムで、これにより以下が可能：

1. **リソース管理の簡素化**：容量計画の自動処理
2. **コストの最適化**：需要に基づくリソースの動的調整
3. **信頼性の向上**：多様化されたインフラストラクチャ選択
4. **柔軟性の強化**：ハイブリッドクラウドアーキテクチャのサポート

適切なCapacity Provider戦略を選択することで、ECSクラスターの効率と経済性を大幅に向上させることができます。