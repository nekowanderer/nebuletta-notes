# AWS EKS と Ingress の統合

[English](../en/17_aws_eks_and_ingress.md) | [繁體中文](../zh-tw/17_aws_eks_and_ingress.md) | [日本語](../ja/17_aws_eks_and_ingress.md) | [インデックスに戻る](../README.md)

## アーキテクチャ概要

```
              Admin EC2                                   Cluster                                                           
  +-------------------------------+    +------------------------------------------------+                                   
  | +--------+                    |    |  +----Service A----+       +---Service B----+  |                                   
  | | eksctl |                    |    |  | +-----+ +-----+ |       |+-----+ +-----+ |  |                                   
  | +--------+                    |    |  | | Pod | | Pod | |-------|| Pod | | Pod | |  |                                   
  |  |                            |    |  | +-----+ +-----+ |   |   |+-----+ +-----+ |  |                                   
  |  |--create k8s cluster ----------->|  +-----------------+   |   +----------------+  |                                   
  |  |                            |    |                        |                       |                                   
  |  +--create cluster resources ----->|            +--------------------+              |                                   
  |                               |    |            |       Ingress      |              |              +---------+          
  | +---------+                   |    |            +--------------------+              |          ----| AWS ALB |          
  | | kubectl |                   |    |                        |                       |    -----/    +---------+          
  | +---------+                   |    |           +------------------------+          -----/               ^               
  |  |                            |    |           | +--------------------+ |    -----/ |                   |               
  |  |--access k8s cluster ----------->|           | | AWS Load Balancer  | |---/       |                   |  able to create               
  |                               |    |           | | Ingress Controller | |           |                   |               
  +-------------------------------+    |           | +--------------------+ |           |              +----------+         
                                       |           | +--------------------+ |           |              | IAM ROLE |         
                                       |           | |   Service Account  | |           |              +----------+         
                                       |           | +--------------------+ |           |                   |               
                                       |           +------------------------+           |                   |               
                                       |                        |                       |                   |               
                                       |         +-----------------------------+        |        +-----------------------+  
                                       |         | OpenID Connect Provider URL |-----------------| IAM Identity Provider |  
                                       |         +-----------------------------+        |        +-----------------------+  
                                       +------------------------------------------------+                                   
```

## アーキテクチャ構成要素

### 管理層（Admin EC2）

#### eksctl ツール
- **目的**: EKS クラスターの作成と管理
- **主な機能**:
  - Kubernetes クラスターの作成
  - クラスターリソースの設定（IAM ロール、セキュリティグループなど）
  - ノードグループの管理

#### kubectl ツール
- **目的**: Kubernetes クラスター管理ツール
- **主な機能**:
  - クラスター内のリソースにアクセスして管理
  - アプリケーションのデプロイ
  - クラスターステータスの監視

### EKS クラスター内部

#### アプリケーションサービス層
- **Service A/B**: 複数の Pod を含むサービス
- **負荷分散**: Kubernetes の負荷分散メカニズムを使用したサービス内部の負荷分散

#### Ingress リソース
- **機能**: 外部トラフィックがクラスターに入る方法を定義
- **特徴**: ルーティングルールの設定、SSL/TLS 終端、ホスト名とパスベースのルーティング

## 権限管理アーキテクチャ

### AWS EKS とローカル K8s の最大の違い

#### OpenID Connect Provider URL
- EKS が自動的に OIDC Provider URL を生成
- K8s クラスターに認識可能な「アイデンティティ」を提供し、K8s 内のコンポーネントが AWS に権限（IAM ロール）を申請できるようにする

#### IAM Identity Provider + Role
1. **IAM Identity Provider の作成**: EKS の OIDC Provider にバインド
2. **IAM ロールの作成**: このロールにロードバランサー（ALB）の作成権限を付与
3. **Service Account のバインド**: Ingress Controller がこのロールを assume し、ALB 作成権限を取得できるようにする

## Ingress Controller の動作メカニズム

### AWS Load Balancer Controller
- **デプロイ方法**: K8s 内に AWS Load Balancer Ingress Controller をデプロイ
- **監視対象**: この Controller は K8s の Ingress リソースを監視
- **自動化**: Ingress を作成/更新すると、Controller が自動的に AWS で ALB を作成

### Service Account と IAM ロールの連携
- Ingress Controller は AWS リソースを操作する権限が必要
- Service Account と IAM ロールを組み合わせて実現
- 安全な権限管理メカニズムを提供

## トラフィックフロー

### 外部トラフィック入力プロセス
1. **ユーザーリクエスト** → AWS ALB
2. **ALB ルーティング** → Ingress ルール（Host/Path）に基づいて対応する K8s Service に転送
3. **Service 負荷分散** → ターゲット Pod

### ALB の特徴
- **レイヤー**: L7（アプリケーション層）
- **機能**: Host/Path による分散が可能
- **利点**: NLB よりも柔軟性が高い

## Service Account 詳細説明

### Service Account とは？
- **定義**: Kubernetes の「マシン用アカウント」
- **目的**: Pod 内のアプリケーションに独自の「アイデンティティ」を提供
- **デフォルト**: 各 Pod にはデフォルトの Service Account が存在

### 主な用途
- **Pod アクセス制御**: Pod がアクセスできる K8s API を管理
- **権限分離**: セキュリティのため、異なる権限を分離して管理

### AWS IAM との関係

#### バインディングメカニズム
- Service Account を AWS IAM ロールにバインド可能
- Pod が特定の SA で起動すると、自動的に対応する IAM ロールの権限を取得
- **利点**: コードや環境変数に Access Key/Secret Key を記述する必要がない

#### 実際のバインディング手順
1. **IAM ロールの作成**: EKS OIDC Provider を信頼
2. **Service Account の作成**: metadata に annotation を追加して IAM ロール ARN をバインド
3. **自動認証情報発行**: AWS が自動的に Pod に一時的な認証情報を発行

#### 設定例
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: alb-ingress-controller
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/ALBIngressControllerRole
```

### 重要概念まとめ
- **Service Account** = K8s の「マシン用」アカウント
- **IAM ロール** = AWS サービス認可のロール
- **両者の結合** = K8s 上のアプリケーションが安全に AWS リソース操作権限を取得

## AWS Load Balancer Controller 動作詳細

### Controller 自身も Pod
- **デプロイ方法**: K8s Deployment、1つまたは複数の Pod を含む
- **権限要件**: AWS ALB リソースを操作する権限が必要
- **解決策**: カスタム Service Account と IAM ロールをバインド

### 完全な動作フロー
1. **変更監視**: Controller が Ingress リソースの変更を検出
2. **リソース作成**: IAM ロール権限を通じて、AWS 上で ALB を自動作成/更新
3. **トラフィックルーティング**: ALB がトラフィックを受信し、Ingress ルールに基づいて異なる Service に分散

## まとめ

### ローカル K8s との主な違い
- **権限連携**: すべてのクロスサービスリソース（ALB など）は事前に IAM 設定が必要
- **自動化レベル**: K8s Ingress Controller が AWS ALB を自動作成・管理可能
- **利点**: ALB の高可用性と運用上の利点を享受

### 核心価値
- **セキュリティ**: IAM ロールによる権限管理で、認証情報のハードコーディングを回避
- **自動化**: 手動設定作業を削減
- **統合性**: AWS サービスとの深度統合