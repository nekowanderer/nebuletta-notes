# Kubernetes Ingress 入門ガイド

[English](../en/26_k8s_ingress_intro.md) | [繁體中文](../zh-tw/26_k8s_ingress_intro.md) | [日本語](../ja/26_k8s_ingress_intro.md) | [インデックスに戻る](../README.md)

```
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                                Cluster                                  │
  │  ┌─────────────┐                                                        │
  │  │  Service A  │                                                        │
  │  │ ┌───┐ ┌───┐ │    all.demo.com/beta                                   │
  │  │ │Pod│ │Pod│ │ ←─────┐                                                │
  │  │ └───┘ └───┘ │       │                                                │
  │  └─────────────┘       │                                                │
  │                        │                                                │
  │                        │    ┌─────────┐   ┌────────────────────┐        │
  │                        └─── │         │   │                    │        │
  │                             │ Ingress │ ← │ Ingress Controller │        │
  │                        ┌─── │         │   │                    │        │
  │                        │    └─────────┘   └────────────────────┘        │
  │                        │                           ↑                    │
  │  ┌─────────────┐       │                           │                    │
  │  │  Service B  │       │                  ┌─────────────────┐           │
  │  │ ┌───┐ ┌───┐ │ ←─────┘                  │  Load Balancer  │ ←── User  │
  │  │ │Pod│ │Pod│ │    all.demo.com/prod     └─────────────────┘           │
  │  │ └───┘ └───┘ │                                                        │
  │  └─────────────┘                                                        │
  └─────────────────────────────────────────────────────────────────────────┘
```

### Kubernetes Ingress とは

Kubernetes Ingress は、クラスタ内のサービスへの外部アクセスを管理するAPIオブジェクトです。HTTPおよびHTTPSトラフィックの単一エントリポイントとして機能し、ホスト名、パス、その他の条件に基づいてトラフィックを異なるサービスにルーティングするルールを提供します。

### 基本的な Ingress アーキテクチャ

基本的な Ingress 設定は、いくつかの重要なコンポーネントで構成されています：

- **Ingress リソース**：ルーティングルールを定義
- **Ingress Controller**：実際のルーティングロジックを実装
- **ロードバランサー**：トラフィックの外部エントリポイント
- **サービス**：ルーティングされたトラフィックを受信するバックエンドサービス

Ingress 導入前（各サービスが独自の Load Balancer を持つ）：
```
Internet
    |
    v
[Load Balancer A] -> [Service A]
[Load Balancer B] -> [Service B]
[Load Balancer C] -> [Service C]
```

Ingress 導入後：
```
External User → Load Balancer → Ingress Controller → Ingress → Service → Pod
```

### デフォルトバックエンドを使用した単一サービスルーティング

最もシンプルな構成では、Ingress はすべてのトラフィックを単一のサービスに転送できます：

- 外部リクエストがロードバランサーを通じて入ってくる
- Ingress Controller がリクエストを処理
- すべてのトラフィックが**デフォルトバックエンド**サービスにルーティングされる
- サービスが関連するPod（Service A に複数のPod）にトラフィックを転送

この設定は、複雑なルーティングルールを必要としないシンプルなアプリケーションに適しています。

### ホストベースのマルチサービスルーティング

Ingress は異なるホスト名に基づいて異なるサービスにルーティングできます：

- **all.demo.com/beta** は Service A にルーティング
- **all.demo.com/prod** は Service B にルーティング

各サービスは独自のPodセットを維持し、以下を可能にします：
- 環境の分離（ベータ版 vs 本番版）
- サービスの独立したスケーリング
- 同一ドメイン内でのパスベースルーティング

### Ingress Controller オプション

**クラウドプロバイダーソリューション：**
- **AWS ALB**（Application Load Balancer）
- **GCP Load Balancer** 
- **Azure Application Gateway**

**セルフマネージドソリューション：**
- **Local Nginx** - 最も一般的なオープンソースオプション

Ingress Controller の選択は以下に依存します：
- インフラストラクチャプラットフォーム（クラウド vs オンプレミス）
- 機能要件（SSL終端、認証など）
- パフォーマンスとコストの考慮事項

### Ingress vs Ingress Controller

| 項目 | Ingress Controller（警備員） | Ingress（入館カード発行規則） | 
|------|------------------------------|------------------------------|
| **本質** | Kubernetes で動作するプログラム、通常は Pod/Deployment | Kubernetes のリソースの一種 | 
| **機能** | Ingress ルールを実際に実行するコンポーネント | ルーティングルールとポリシーを定義 |
| **比喩** | 「警備員」のように入館カードをチェックしてルールを実行 | 「入館カード発行規則」のように誰がどのエリアに入れるかを規定 |

#### 両者の関係
- **両方不可欠**：Ingress Controller と Ingress は両方存在しないと機能しない
- **明確な分業**：Ingress がルールを定義し、Controller がルールを実行
- **動的連携**：Controller は Ingress の変更を監視してリアルタイムで更新

### 動作プロセス（NGINX Ingress Controller の例）

#### 1. 起動時に Ingress Controller として自身を登録
例えば、Helm や YAML を使用して `nginx-ingress-controller` の Deployment + Service のセットをデプロイします。

この Pod セットは自動的に K8S API Server に Ingress Controller として登録されます。

#### 2. Ingress リソースの変更を継続的に監視
Ingress Controller は K8S のすべての Namespace にある Ingress オブジェクトを継続的に Watch（監視）します（関連する Service/Secret も監視）。

誰かが Ingress を追加、変更、削除するたびに、Controller は通知を受けます。

#### 3. Ingress ルールを解析
Controller は Ingress リソースの内容を読み取ります。例えば：
- どのホスト（ドメイン）を処理するか
- どのパスをどの Service/Port に導くか
- SSL/TLS を追加するか（証明書はどこにあるか）

#### 4. リバースプロキシ設定ファイルを自動生成
NGINX Controller の例では、K8S Ingress の内容を自身の NGINX 設定ファイル（nginx.conf）に「翻訳」します。

ルールが変更されると、Controller は設定を動的にリロードし、Pod を手動で再起動する必要はありません。

#### 5. 実際に外部リクエストを受信してプロキシ
外部の HTTP/HTTPS トラフィックはロードバランサーや NodePort を通って Ingress Controller の Service に導かれます。

NGINX は翻訳されたルールに従って、リクエストを対応する K8S Service にルーティングします（その後 Service が Pod に送信）。

### Ingress 使用の主な利点

- **単一エントリポイント**：一つのインターフェースを通じて外部アクセスを統合
- **コスト効率**：複数のLoadBalancerサービスの必要性を削減
- **高度なルーティング**：ホストベースおよびパスベースのルーティングをサポート
- **SSL終端**：HTTPS証明書を一元的に処理可能
- **負荷分散**：様々な負荷分散アルゴリズムをサポート