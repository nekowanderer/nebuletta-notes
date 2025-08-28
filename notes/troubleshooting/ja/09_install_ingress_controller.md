# Helm を使用した Ingress Controller インストールの経験共有

[English](../en/09_install_ingress_controller.md) | [繁體中文](../zh-tw/09_install_ingress_controller.md) | [日本語](09_install_ingress_controller.md) | [インデックスに戻る](../README.md)

---

## 背景
- 実験日: 2025/08/28
- 難易度：🤬
- 説明: 元々 `kubectl apply -f deploy.yaml` を使用して ingress-nginx controller をインストールする際に、バージョンが古い、設定が複雑などの問題に遭遇しました。Helm インストールに切り替えることで、プロセスが大幅に簡素化されました。

## 重要な概念の明確化

Kubernetes では、2つの異なる概念を区別する必要があります：

- **Ingress Controller**: トラフィックを処理する実際のプログラム（nginx など）、Ingress ルールを実装する責任を持つ
- **Ingress**: ルーティングルールを定義する Kubernetes リソース、Controller にトラフィックの転送方法を指示する

Helm がインストールするのは **Ingress Controller** で、その後 **Ingress** リソースを別途作成してルーティングルールを定義する必要があります。

---

## 遭遇した問題

- 公式提供の `deploy.yaml` ファイルを使用して ingress-nginx controller をインストール
- インストール後、バージョンが古く、機能が不完全で、Pod が再起動し続けることを発見
- Webhook、Certificate、RBAC などの複雑な設定を手動で処理する必要
- 更新のたびに新しい YAML ファイルを再ダウンロードする必要
- 削除時に複数のリソースを手動で削除する必要

## トラブルシューティングプロセス

### 第1段階：従来のインストール方法を試行

```bash
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.44.0/deploy/static/provider/cloud/deploy.yaml
```

**遭遇した問題：**
- バージョンが固定され、簡単にアップグレードできない
- YAML ファイルに手動設定されたリソースが大量に含まれている
- namespace、service account、cluster role などを手動で処理する必要
- 削除時にすべての関連リソースを手動で削除する必要

### 第2段階：Helm インストールに切り替え

Helm が上記のすべての問題を解決できることを発見し、Helm インストールに切り替えることを決定。

## 解決策

### ステップ1：Helm を使用して Ingress Controller をインストール

#### 1.1 Helm Repository を追加
```bash
$ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```

#### 1.2 Repository を更新
```bash
$ helm repo update
```

#### 1.3 Ingress Controller をインストール
```bash
$ helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

### カスタマイズインストール（オプション）

デフォルトの LoadBalancer の代わりに NodePort を使用する必要がある場合：

```bash
$ helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.http=30080 \
  --set controller.service.nodePorts.https=30443
```

### ステップ2：リソースを作成

Ingress Controller をインストールした後、ルーティングルールを定義するために Ingress リソースを作成する必要があります：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-ingress
  annotations:
    # We are defining this annotation to prevent nginx
    # from redirecting requests to `https` for now
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: nginx # This line is necessary otherwise the ingress can not expose the address
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-svc
                port:
                  number: 1234
          - path: /hello
            pathType: Prefix
            backend:
              service:
                name: hellok8s-svc
                port:
                  number: 4567
```

次に Service：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    app: nginx
  ports:
  - port: 1234
    targetPort: 80

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx-container

```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hellok8s-svc
spec:
  selector:
    app: hellok8s
  ports:
  - port: 4567
    targetPort: 4567

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: hellok8s
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hellok8s
  template:
    metadata:
      labels:
        app: hellok8s
    spec:
      containers:
      - image: brianstorti/hellok8s:v3
        name: hellok8s-container
```
順番に apply してください。

## インストールの検証

### Ingress Controller の状態を確認

#### Pod の状態を確認
```bash
$ kubectl get pods -n ingress-nginx
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-66cdbcf9bf-g995j   1/1     Running   0          44m
```

#### Service の状態を確認
```bash
$ kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP        EXTERNAL-IP     PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   192.168.194.148   192.168.139.2   80:31802/TCP,443:31809/TCP   44m
ingress-nginx-controller-admission   ClusterIP      192.168.194.149   <none>          443/TCP                      44m
```

### Ingress リソースの状態を確認
```bash
$ kubectl get ingress
NAME            CLASS   HOSTS   ADDRESS         PORTS   AGE
hello-ingress   nginx   *       192.168.139.2   80      42m
```

### 接続をテスト
```bash
curl http://192.168.139.2/hello
```

## Ingress Controller のバージョンを確認

### 方法1：Helm を使用して Release 情報を確認
```bash
# release の詳細情報を確認
$ helm list -n ingress-nginx
NAME         	NAMESPACE    	REVISION	UPDATED                             	STATUS  	CHART               	APP VERSION
ingress-nginx	ingress-nginx	1       	2025-08-28 21:13:57.830894 +0800 CST	deployed	ingress-nginx-4.13.1	1.13.1

# release の詳細状態を確認
$ helm status ingress-nginx -n ingress-nginx
NAME: ingress-nginx
LAST DEPLOYED: Thu Aug 28 21:13:57 2025
NAMESPACE: ingress-nginx
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The ingress-nginx controller has been installed.
It may take a few minutes for the load balancer IP to be available.
You can watch the status by running 'kubectl get service --namespace ingress-nginx ingress-nginx-controller --output wide --watch'

An example Ingress that makes use of the controller:
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: example
    namespace: foo
  spec:
    ingressClassName: nginx
    rules:
      - host: www.example.com
        http:
          paths:
            - pathType: Prefix
              backend:
                service:
                  name: exampleService
                  port:
                    number: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
      - hosts:
        - www.example.com
        secretName: example-tls

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls
```

### 方法2：Chart バージョン情報を確認
```bash
# chart の metadata を確認
$ helm get metadata ingress-nginx -n ingress-nginx
NAME: ingress-nginx
CHART: ingress-nginx
VERSION: 4.13.1
APP_VERSION: 1.13.1
ANNOTATIONS: artifacthub.io/changes=- 'Make: Add `helm-test` target. (#13659)'
- Update Ingress-Nginx version controller-v1.13.1
,artifacthub.io/prerelease=false
DEPENDENCIES:
NAMESPACE: ingress-nginx
REVISION: 1
STATUS: deployed
DEPLOYED_AT: 2025-08-28T21:13:57+08:00
```

### 方法3：Pod 内のバージョン情報を確認
```bash
# ingress controller pod の詳細情報を確認
$ kubectl describe pod -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# または pod の image タグを直接確認
$ kubectl get pods -n ingress-nginx -o jsonpath='{.items[0].spec.containers[0].image}'
registry.k8s.io/ingress-nginx/controller:v1.13.1@sha256:37e489b22ac77576576e52e474941cd7754238438847c1ee795ad6d59c02b12a%
```

### 方法4：利用可能な Chart バージョンを確認
```bash
# リモート repository で利用可能なバージョンを確認
$ helm search repo ingress-nginx/ingress-nginx --versions
NAME                       	CHART VERSION	APP VERSION	DESCRIPTION
ingress-nginx/ingress-nginx	4.13.1       	1.13.1     	Ingress controller for Kubernetes using NGINX a...
ingress-nginx/ingress-nginx	4.13.0       	1.13.0     	Ingress controller for Kubernetes using NGINX a...
ingress-nginx/ingress-nginx	4.12.5       	1.12.5     	Ingress controller for Kubernetes using NGINX a...
ingress-nginx/ingress-nginx	4.12.4       	1.12.4     	Ingress controller for Kubernetes using NGINX a...
...

# ローカル chart 情報を確認
$ helm show chart ingress-nginx/ingress-nginx
```

## なぜ Helm が優れているのか？

| 項目 | kubectl apply | Helm |
|------|---------------|------|
| バージョン管理 | 新しいバージョンを手動でダウンロードする必要 | バージョンを指定でき、自動処理 |
| 設定の複雑さ | すべてのリソースを手動で処理する必要 | Webhook、Certificate、RBAC を自動処理 |
| カスタマイズ | YAML を修正する必要 | `--set` パラメータを使用 |
| 削除 | 複数のリソースを手動で削除する必要 | `helm uninstall` で一度に完了 |
| アップグレード | 再ダウンロードして適用する必要 | `helm upgrade` で自動処理 |

## まとめ

元々 `kubectl apply -f deploy.yaml` を使用する方法は直接的でしたが、バージョン管理や設定の複雑さの問題に遭遇しました。Helm に切り替えた後、インストールプロセス全体がより簡潔で保守しやすくなり、特に頻繁な更新やカスタマイズが必要な環境に適しています。

## 重要な注意事項

**Helm がインストールするのは Ingress Controller であり、Ingress 自体ではありません**。Controller をインストールした後、まだ以下が必要です：

1. ルーティングルールを定義するために Ingress リソースを作成
2. Ingress リソース内の `ingressClassName` が Controller と一致することを確認
3. 実際のビジネスロジックを処理するための対応する Service と Pod を作成

このアーキテクチャにより、Ingress Controller と Ingress ルールが分離され、より良い柔軟性と保守性が提供されます。
