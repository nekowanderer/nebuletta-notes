# Kubernetes Path-based Ingress 実装ガイド

[English](../en/30_k8s_path_based.md) | [繁體中文](../zh-tw/30_k8s_path_based.md) | [日本語](../ja/30_k8s_path_based.md) | [インデックスに戻る](../README.md)

## 概要

Path-based Ingress は Kubernetes における重要なトラフィックルーティング方法で、URL パスに基づいてリクエストを異なるバックエンドサービスに転送することができます。本ガイドでは、Minikube 環境で Path-based Ingress を実装し、`/beta` と `/prod` パスをそれぞれ異なるアプリケーションサービスにルーティングする方法を示します。

## 環境セットアップ

### Docker サービスの開始

```bash
$ sudo service docker start
$ docker ps
```

### Minikube クラスターの作成

```bash
$ minikube start --driver docker
$ minikube status
```

### Ingress アドオンの有効化

```bash
$ minikube addons enable ingress
$ kubectl get all -n ingress-nginx
```

## アプリケーションのデプロイ

### Beta アプリケーションのデプロイ

```bash
$ kubectl apply -f beta-app-all.yaml
$ kubectl get all -n app-ns
```

### Production アプリケーションのデプロイ

```bash
$ kubectl apply -f prod-app-all.yaml
$ kubectl get all -n app-ns
```

## Ingress の設定

### Path-based Ingress 設定ファイル

`ingress-path.yaml` ファイルを作成：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-path
  namespace: app-ns
spec:
  ingressClassName: nginx
  rules:
  - host: all.demo.com
    http:
      paths:
      - pathType: Prefix
        path: /beta
        backend:
          service:
            name: beta-app-service-clusterip
            port:
              number: 8080
      - pathType: Prefix
        path: /prod
        backend:
          service:
            name: prod-app-service-clusterip
            port:
              number: 8080
```

### Ingress のデプロイ

```bash
$ kubectl apply -f ingress-path.yaml

$ kubectl get ingress -n app-ns
NAME               CLASS   HOSTS                         ADDRESS        PORTS   AGE
ingress-hostname   nginx   beta.demo.com,prod.demo.com   192.168.49.2   80      26m
ingress-path       nginx   all.demo.com,prod.demo.com    192.168.49.2   80      63s
```

## 設定の説明

### Path-based ルーティングルール

- **`/beta` パス**：`beta-app-service-clusterip` サービスに転送
- **`/prod` パス**：`prod-app-service-clusterip` サービスに転送
- **`pathType: Prefix`**：前置詞マッチングを使用、`/beta` は `/beta`、`/beta/`、`/beta/api` などのパスにマッチ

### 主要コンポーネント

- **`ingressClassName: nginx`**：NGINX Ingress Controller の使用を指定
- **`host: all.demo.com`**：仮想ホスト名を設定
- **`pathType: Prefix`**：パスマッチングタイプは前置詞マッチング

## テストと検証

### 1. Ingress IP の取得

```bash
$ kubectl get ingress -n app-ns
```

### 2. Beta アプリケーションのテスト

```bash
$ curl [ingress_ip]:80/beta -H 'Host: all.demo.com'
```

### 3. Production アプリケーションのテスト

```bash
$ curl [ingress_ip]:80/prod -H 'Host: all.demo.com'
```

## リソースのクリーンアップ

テスト完了後、すべてのリソースをクリーンアップ：

```bash
$ kubectl delete ns app-ns
```

## 注意事項

1. **パスの優先順位**：Ingress コントローラーは設定順序でパスをマッチングするため、より長いパスをより短いパスの前に配置する必要があります
2. **Host ヘッダー**：テスト時は正しい Host ヘッダーを含める必要があり、そうでなければリクエストが正しくルーティングされない可能性があります
3. **名前空間**：すべてのリソースが同じ名前空間にあることを確認するか、名前空間を正しく指定してください
4. **サービスの可用性**：Ingress をデプロイする前に、バックエンドサービスが正常に動作していることを確認してください

## よくある質問

Q: なぜ Host ヘッダーの設定が必要なのですか？
A: Ingress ルールで `host: all.demo.com` が設定されているため、テスト時にこの Host ヘッダーを含めてルールに正しくマッチさせる必要があります。

Q: より多くのパスを追加するにはどうすればよいですか？
A: `paths` 配列に新しいパス設定を追加し、`path`、`pathType`、対応するバックエンドサービスを指定します。

Q: 異なるパスタイプを設定するにはどうすればよいですか？
A: `pathType` は以下に設定できます：
- `Prefix`：前置詞マッチング
- `Exact`：完全一致
- `ImplementationSpecific`：実装固有のマッチング