# Kubernetes Ingress 総まとめ

[English](../en/31_k8s_ingress_summary.md) | [繁體中文](../zh-tw/31_k8s_ingress_summary.md) | [日本語](../ja/31_k8s_ingress_summary.md) | [インデックスに戻る](../README.md)

## 概要

このノートでは、Kubernetes Ingress の3つの主要な設定方法をまとめます：Default Backend、Hostname-based、Path-based。それぞれの方法には特定の使用ケースと設定アプローチがあります。

## 3つの設定方法の比較

### 1. Default Backend（デフォルトバックエンド）

#### 使用ケース
- **404 エラー処理**：リクエストがどのルールにもマッチしない場合のデフォルト応答を提供
- **メンテナンスページ**：システムメンテナンス中に統一されたメンテナンスページを表示
- **エラー処理**：統一されたエラーページや API レスポンスを提供
- **シンプルなアプリケーション**：単一サービスのみを外部に公開する場合

#### YAML 設定例
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-defaultbackend
  namespace: app-ns
spec:
  ingressClassName: nginx
  defaultBackend:
    service:
      name: default-app-service
      port:
        number: 8080
```

#### 特徴
- **最もシンプルな設定**：`defaultBackend` の指定のみが必要
- **グローバルデフォルト**：すべてのマッチしないリクエストはデフォルトサービスに転送
- **ルーティングルールなし**：`rules` セクションを含まない
- **単一エントリポイント**：単一アプリケーションやシンプルなアーキテクチャに適している

### 2. Hostname-based Routing（ホスト名ベースルーティング）

#### 使用ケース
- **マルチテナントアーキテクチャ**：異なる顧客が異なるドメイン名を使用
- **環境分離**：beta.demo.com、prod.demo.com など
- **マイクロサービスアーキテクチャ**：各サービスが独立したドメイン名を持つ
- **ブランド分離**：異なるブランドが異なるドメイン名を使用

#### YAML 設定例
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-hostname
  namespace: app-ns
spec:
  ingressClassName: nginx
  rules:
  - host: beta.demo.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: beta-app-service
            port:
              number: 8080
  - host: prod.demo.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: prod-app-service
            port:
              number: 8080
```

#### 特徴
- **ドメイン分離**：異なるドメイン名が異なるサービスを指す
- **明確な分離**：各環境またはサービスが独立したドメイン名を持つ
- **管理の容易さ**：各ドメイン名に異なる SSL 証明書を設定可能
- **マルチ環境に適している**：開発、テスト、本番環境の分離

### 3. Path-based Routing（パスベースルーティング）

#### 使用ケース
- **単一ドメイン複数サービス**：同一ドメイン名下で複数のサービスを提供
- **API ゲートウェイ**：統一エントリポイント、パスによって異なる API にルーティング
- **フロントエンドルーティング**：SPA アプリケーションのバックエンド API ルーティング
- **バージョン管理**：異なるバージョンの API が異なるパスを使用

#### YAML 設定例
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-path
  namespace: app-ns
spec:
  ingressClassName: nginx
  rules:
  - host: api.demo.com
    http:
      paths:
      - pathType: Prefix
        path: /beta
        backend:
          service:
            name: beta-app-service
            port:
              number: 8080
      - pathType: Prefix
        path: /prod
        backend:
          service:
            name: prod-app-service
            port:
              number: 8080
      - pathType: Prefix
        path: /api/v1
        backend:
          service:
            name: api-v1-service
            port:
              number: 8080
```

#### 特徴
- **パス分離**：同一ドメイン名下でパスによってルーティング
- **統一エントリ**：すべてのサービスが同一ドメイン名を共有
- **柔軟な設定**：複数のパスタイプを組み合わせ可能
- **API 設計に適している**：RESTful API の標準的な手法

## 設定差異の比較

### YAML 構造の違い

| 設定方法 | 主要セクション | ルーティングルール | バックエンド設定 |
|---------|--------------|------------------|-----------------|
| Default Backend | `defaultBackend` | なし | 単一サービス |
| Hostname-based | `rules[].host` | ドメイン名による | ドメインごとに1つのサービス |
| Path-based | `rules[].http.paths` | パスによる | パスごとに1つのサービス |

### ルーティング優先順位

1. **Hostname-based**：ドメイン名マッチングが優先
2. **Path-based**：同一ドメイン名下でのパスマッチング
3. **Default Backend**：すべてのルールがマッチしない場合に使用

### テスト方法の違い

#### Default Backend
```bash
$ curl [ingress_ip]:80
```

#### Hostname-based
```bash
$ curl [ingress_ip]:80 -H 'Host: beta.demo.com'
$ curl [ingress_ip]:80 -H 'Host: prod.demo.com'
```

#### Path-based
```bash
$ curl [ingress_ip]:80/beta -H 'Host: api.demo.com'
$ curl [ingress_ip]:80/prod -H 'Host: api.demo.com'
```

## 選択の推奨事項

### Default Backend を選択する場合：
- 単一サービスのみを外部に公開する必要がある
- 統一されたエラー処理メカニズムが必要
- システムアーキテクチャがシンプルで、複雑なルーティングが不要

### Hostname-based を選択する場合：
- 異なる環境や顧客に対して独立したドメイン名を提供する必要がある
- 異なるドメイン名に対して異なる SSL 証明書を設定する必要がある
- 明確なサービス分離が必要

### Path-based を選択する場合：
- 単一ドメイン名で複数のサービスを提供したい
- RESTful API アーキテクチャを設計している
- バージョン管理や環境分離が必要だが、ドメイン名を共有したい

### 組み合わせ使用
実際のアプリケーションでは、これら3つの方法を組み合わせて使用できます：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-combined
  namespace: app-ns
spec:
  ingressClassName: nginx
  defaultBackend:
    service:
      name: error-page-service
      port:
        number: 8080
  rules:
  - host: api.demo.com
    http:
      paths:
      - pathType: Prefix
        path: /v1
        backend:
          service:
            name: api-v1-service
            port:
              number: 8080
      - pathType: Prefix
        path: /v2
        backend:
          service:
            name: api-v2-service
            port:
              number: 8080
  - host: admin.demo.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: admin-service
            port:
              number: 8080
```

## 注意事項

1. **パス優先順位**：Path-based 設定では、より長いパスをより短いパスの前に配置する必要があります
2. **Host ヘッダー**：テスト時は正しい Host ヘッダーを含める必要があります
3. **SSL 証明書**：Hostname-based では各ドメイン名に異なる SSL 証明書を設定できます
4. **名前空間**：すべての関連リソースが正しい名前空間にあることを確認してください
5. **サービス可用性**：Ingress をデプロイする前に、バックエンドサービスが正常に動作していることを確認してください

## まとめ

Kubernetes Ingress の3つの設定方法はそれぞれ適用可能なシナリオがあります：

- **Default Backend**：シンプルなアーキテクチャとエラー処理に適している
- **Hostname-based**：マルチテナントとマルチ環境分離に適している
- **Path-based**：API ゲートウェイと統一エントリデザインに適している

実際のアプリケーションでは、要件に応じてこれらの方法を組み合わせて、柔軟で拡張可能なトラフィックルーティングアーキテクチャを構築できます。