# AWS ロードバランシングガイド

[English](../en/19_aws_load_balancing.md) | [繁體中文](../zh-tw/19_aws_load_balancing.md) | [日本語](../ja/19_aws_load_balancing.md) | [インデックスに戻る](../README.md)

### AWS ロードバランサータイプの比較

AWSでは3つの主要なロードバランサータイプを提供しており、それぞれに特定の使用シナリオと機能特性があります：

| タイプ | 適用シナリオ | サポートプロトコル | 動作レイヤー | 主な特徴 |
|------|----------|----------|----------|----------|
| **Application Load Balancer (ALB)** | HTTP/HTTPSアプリケーション | HTTP、HTTPS、WebSocket | Layer 7（アプリケーション層） | パスルーティング、ホストルーティング、Lambda統合 |
| **Network Load Balancer (NLB)** | 高性能TCP/UDPトラフィック | TCP、UDP、TLS | Layer 4（トランスポート層） | 超低遅延、静的IP、極高スループット |
| **Gateway Load Balancer (GLB)** | サードパーティネットワーク機器 | GENEVE | Layer 3（ネットワーク層） | ファイアウォール、IDS/IPS、DPI機器統合 |

### ロードバランサーのコア機能

#### トラフィック配分アルゴリズム
- **Round Robin**：各ターゲットに順次リクエストを配分
- **Least Outstanding Requests**：処理中リクエスト数が最も少ないターゲットに配分
- **Flow Hash**：送信元IP、宛先IP、プロトコルのハッシュ値に基づく配分

#### ヘルスチェック機能
- **ヘルスチェックパス**：ターゲットが正常に応答するかを定期的にチェック
- **正常閾値**：連続成功回数が閾値に達すると正常とマーク
- **異常閾値**：連続失敗回数が閾値に達すると異常とマーク
- **チェック間隔**：ヘルスチェックの頻度設定

---

## Target Group（ターゲットグループ）

### Target GroupとALBの関連

#### 基本関係
- **ALBはEC2やサービスに直接ルーティングできません**
- **Target Groupを仲介として使用する必要があります**
- Target GroupはALBの「ターゲットコンテナ」です

#### 関連アーキテクチャ図
```
Internet → ALB → Listener → Rules → Target Group → Targets (EC2/IP/Lambda)
```

#### ALB ListenerとTarget Groupの統合
```bash
# ALBのListenerはTarget Groupにトラフィックを転送します
$ aws elbv2 create-listener \
    --load-balancer-arn arn:aws:elasticloadbalancing:region:account:loadbalancer/app/my-alb/1234567890123456 \
    --protocol HTTP \
    --port 80 \
    --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:region:account:targetgroup/my-targets/1234567890123456
```

#### 1つのALBに複数のTarget Groupを使用する場合
- **パスルーティング**：`/api/*` → API Target Group
- **ホストルーティング**：`admin.example.com` → Admin Target Group
- **デフォルトルーティング**：その他すべてのリクエスト → Default Target Group

### 実際の応用例

#### マイクロサービスアーキテクチャ
```
ALB
├── /api/users/* → User Service Target Group
├── /api/orders/* → Order Service Target Group  
├── /admin/* → Admin Panel Target Group
└── /* → Frontend Target Group
```

#### ブルーグリーンデプロイメント
```
ALB
├── 90% トラフィック → Production Target Group（ブルー）
└── 10% トラフィック → Staging Target Group（グリーン）
```

### ロードバランシングの流れ

1. **ALBがリクエストを受信**
2. **Listenerがポート番号とプロトコルをチェック**
3. **RulesがどのTarget Groupに転送するかを決定**
4. **Target Groupがアルゴリズムに基づいて健全なTargetを選択**
5. **選択されたTargetにリクエストを転送**

### ALBにおけるTarget Groupの役割

| 機能 | Target Groupの役割 |
|------|-------------------|
| **ヘルスチェック** | バックエンドサービスの状態を継続的に監視 |
| **負荷分散** | アルゴリズムに基づいてトラフィックを分散 |
| **サービスディスカバリー** | ターゲットの動的登録/登録解除 |
| **セッション粘性** | ユーザーセッションの一貫性を維持 |

### ターゲットグループのタイプ

#### Instanceターゲットグループ
- **ターゲットタイプ**：EC2インスタンス
- **登録方法**：EC2インスタンスIDを直接指定
- **適用シナリオ**：従来のEC2アーキテクチャ、Auto Scaling Groupとの統合が必要

#### IPターゲットグループ
- **ターゲットタイプ**：IPアドレス
- **登録方法**：IPアドレスとポート番号を直接指定
- **適用シナリオ**：コンテナ化アプリケーション、マイクロサービスアーキテクチャ、クロスVPC通信

#### Lambdaターゲットグループ
- **ターゲットタイプ**：Lambda関数
- **登録方法**：Lambda関数ARNを指定
- **適用シナリオ**：サーバーレスアーキテクチャ、イベント駆動型HTTP API

### ターゲットグループのヘルスチェック設定

```bash
# ヘルスチェック設定例
$ aws elbv2 modify-target-group \
    --target-group-arn arn:aws:elasticloadbalancing:region:account:targetgroup/my-targets/1234567890123456 \
    --health-check-path /health \
    --health-check-interval-seconds 30 \
    --health-check-timeout-seconds 5 \
    --healthy-threshold-count 2 \
    --unhealthy-threshold-count 5
```

### ターゲットグループ属性の設定

| 属性 | 説明 | デフォルト値 | 推奨設定 |
|------|------|--------|----------|
| **deregistration_delay.timeout_seconds** | 登録解除待機時間 | 300秒 | アプリケーションの終了時間に合わせて調整 |
| **stickiness.enabled** | セッション粘性を有効化 | false | ステートフルアプリケーションの場合trueに設定 |
| **stickiness.type** | 粘性タイプ | lb_cookie | アプリケーションクッキーまたはロードバランサークッキー |
| **load_balancing.algorithm.type** | ロードバランシングアルゴリズム | round_robin | アプリケーションの特性に応じて選択 |

---

## Trust Stores（信頼ストア）

### 信頼ストアの概念

信頼ストアは、AWS Application Load Balancerの高度な機能で、クライアント証明書の管理と検証に使用されます：

#### 主な機能
- **クライアント証明書検証**：ALBに接続するクライアント証明書を検証
- **mTLSサポート**：相互TLS認証を実装
- **証明書チェーン検証**：完全な証明書チェーンを検証
- **証明書失効チェック**：CRLとOCSP検証をサポート

#### 使用シナリオ
- **企業内部API**：クライアント証明書認証を強制する必要がある
- **B2B統合**：パートナーとの安全なAPI通信
- **コンプライアンス要件**：特定業界のセキュリティ標準を満たす
- **ゼロトラストアーキテクチャ**：エンドツーエンドの証明書検証を実装

### 信頼ストアの設定フロー

#### 1. 信頼ストアの作成
```bash
$ aws elbv2 create-trust-store \
    --name my-trust-store \
    --ca-certificates-bundle-s3-bucket my-ca-bucket \
    --ca-certificates-bundle-s3-key ca-bundle.pem
```

#### 2. リスナーとの関連付け
```bash
$ aws elbv2 modify-listener \
    --listener-arn arn:aws:elasticloadbalancing:region:account:listener/app/my-load-balancer/1234567890123456/1234567890123456 \
    --mutual-authentication Mode=verify,TrustStoreArn=arn:aws:elasticloadbalancing:region:account:truststore/my-trust-store/1234567890123456
```

#### 3. 証明書フォーマット要件
- **フォーマット**：PEM形式のCA証明書
- **サイズ制限**：最大1MB
- **証明書数**：信頼ストアあたり最大10,000個の証明書
- **更新方法**：S3を通じて新しい証明書バンドルをアップロード

### 信頼ストアのベストプラクティス

#### セキュリティ考慮事項
1. **定期的な証明書更新**：証明書ローテーション機構を構築
2. **最小権限の原則**：必要な証明書アクセス権限のみを付与
3. **監視とログ記録**：証明書検証イベントを記録
4. **バックアップ戦略**：証明書と信頼ストア設定を定期的にバックアップ

#### 運用管理
1. **証明書ライフサイクル管理**：証明書の有効期限を追跡
2. **自動化デプロイメント**：Infrastructure as Codeで管理
3. **テスト環境**：テスト用の信頼ストアを構築
4. **エラーハンドリング**：証明書検証失敗時の処理ロジックを設定

---

## ロードバランサーの高度な機能

### ルーティングルール設定

#### ALBルーティングルールタイプ
- **パスルーティング**：URLパスに基づいてトラフィックを分散
- **ホストルーティング**：Hostヘッダーに基づいてトラフィックを分散
- **ヘッダールーティング**：HTTPヘッダーに基づいてトラフィックを分散
- **クエリ文字列ルーティング**：クエリパラメータに基づいてトラフィックを分散

#### ルーティングルール例
```bash
# パスルーティングルールの作成
$ aws elbv2 create-rule \
    --listener-arn arn:aws:elasticloadbalancing:region:account:listener/app/my-load-balancer/1234567890123456/1234567890123456 \
    --conditions Field=path-pattern,Values="/api/*" \
    --priority 100 \
    --actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:region:account:targetgroup/api-targets/1234567890123456
```

### クロスゾーンロードバランシング

#### クロスゾーンロードバランシング設定
- **デフォルト動作**：ALBとGLBはデフォルトで有効、NLBはデフォルトで無効
- **パフォーマンス影響**：クロスゾーントラフィックコストが増加する可能性
- **高可用性**：故障耐性能力を向上

#### コスト最適化の考慮事項
- **ゾーン認識ルーティング**：同じゾーンのターゲットを優先してルーティング
- **トラフィック分析**：クロスゾーントラフィックパターンを監視
- **コスト監視**：クロスゾーントラフィックアラートを設定

### AWSサービスとの統合

#### Auto Scaling統合
- **動的拡張**：負荷に基づいてターゲット数を自動調整
- **ヘルスチェック統合**：不健全なインスタンスを自動的に置換
- **ウォームアップ時間**：新しいインスタンスの準備時間設定

#### CloudWatch監視
- **重要指標**：リクエスト数、遅延、エラー率
- **カスタムアラーム**：パフォーマンスと可用性のアラームを設定
- **ログ分析**：アクセスログをS3に保存して分析

#### WAF統合
- **セキュリティ保護**：Webアプリケーションファイアウォールルール
- **DDoS保護**：AWS Shieldと統合
- **地理的ブロック**：地理的位置に基づいてアクセスを制御

---

## ロードバランサーのベストプラクティス

### パフォーマンス最適化
1. **接続再利用**：Keep-Alive接続を有効化
2. **圧縮設定**：HTTP圧縮を有効化
3. **キャッシュ戦略**：適切なキャッシュヘッダーを設定
4. **SSL終端**：ロードバランサーレベルでSSLを処理

### セキュリティ強化
1. **SSL/TLSポリシー**：最新のセキュリティポリシーを使用
2. **アクセス制御**：セキュリティグループルールを設定
3. **証明書管理**：AWS Certificate Managerを使用
4. **監視ログ**：アクセスログ記録を有効化

### トラブルシューティング
1. **ヘルスチェック失敗**：ターゲットの健康状態をチェック
2. **接続タイムアウト**：タイムアウト設定を調整
3. **502/503エラー**：バックエンドサービスの状態をチェック
4. **SSLハンドシェイク失敗**：証明書設定を確認

### コスト最適化
1. **ロードバランサータイプ選択**：ニーズに応じて適切なタイプを選択
2. **ターゲットグループ設定**：ターゲット数と分散を最適化
3. **使用量監視**：トラフィックパターンを定期的にチェック
4. **予約容量**：Savings Plansの使用を検討