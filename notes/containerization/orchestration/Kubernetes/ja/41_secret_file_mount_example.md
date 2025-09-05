# Secret ファイルマウント実践例

[English](../en/41_secret_file_mount_example.md) | [繁體中文](../zh-tw/41_secret_file_mount_example.md) | [日本語](../ja/41_secret_file_mount_example.md) | [回到索引](../README.md)

## 完全例：TLS証明書とAPI KEY管理

### 1. Secretの作成

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  tls.crt: |
    -----BEGIN CERTIFICATE-----
    MIIDXTCCAkWgAwIBAgIJAKoK/OvD8dO3MA0GCSqGSIb3DQEBCwUAMEUxCzAJBgNV
    BAYTAkFVMRMwEQYDVQQIDApTb21lLVN0YXRlMSEwHwYDVQQKDBhJbnRlcm5ldCBX
    aWRnaXRzIFB0eSBMdGQwHhcNMjMwMTAxMDAwMDAwWhcNMjQwMTAxMDAwMDAwWjBF
    -----END CERTIFICATE-----
  tls.key: |
    -----BEGIN PRIVATE KEY-----
    MIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQC7VJTUt9Us8cKB
    xIuQyC9Wr0LzFWnQhsgJ1pPrBvYkqjBVHq0BYSOqRr6ny+0F8HqyJfS3QyQ2QyQ2
    -----END PRIVATE KEY-----
  api-key: "sk-1234567890abcdef1234567890abcdef"
  db-password: "mySecurePassword123"
```

### 2. Deploymentの作成

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app
        image: nginx:alpine
        ports:
        - containerPort: 443
        - containerPort: 80
        volumeMounts:
        - name: tls-certs
          mountPath: /etc/ssl/certs # tls-certs volumeをコンテナの/etc/ssl/certsディレクトリにマウント
          readOnly: true
        - name: app-secrets
          mountPath: /app/secrets
          readOnly: true
        - name: db-secret
          mountPath: /app/config
          readOnly: true
      volumes:
      - name: tls-certs
        secret:
          secretName: app-secrets
          items:
          - key: tls.crt       # Secret内のkey
            path: server.crt   # volume内で作成するファイル名
          - key: tls.key
            path: server.key
      - name: app-secrets
        secret:
          secretName: app-secrets
          items:
          - key: api-key
            path: api-key.txt
      - name: db-secret
        secret:
          secretName: app-secrets
          items:
          - key: db-password
            path: db-password.txt
```

- 重要なポイント
  - 一対一マッピング
  - 各keyが一つのpathに対応
  - keyはSecret内のフィールド名
  - pathはvolume内のファイル名

- ファイル名の変更
  - Secret内のtls.crt → volume内のserver.crt
  - Secret内のtls.key → volume内のserver.key
  - これにより、ファイル名をアプリケーションの要件により適合させることができる

- 同じSecretを複数のvolumeで使用
  - 同じSecretを複数のvolumeで使用可能
  - 各volumeは異なるkeyとpathを選択可能
  - これにより、異なるタイプのデータを異なるディレクトリにマウント可能

- 柔軟なファイル組織
  - TLS証明書を/etc/ssl/certs/にマウント（標準SSLディレクトリ）
  - API KEYを/app/secrets/にマウント（アプリケーション専用ディレクトリ）
  - データベースパスワードを/app/config/にマウント（設定ディレクトリ）

- 実践的アプリケーションの利点
  - セキュリティ：機密データが環境変数に表示されない
  - 柔軟性：ファイル名を変更してアプリケーション要件に適合可能
  - 組織性：異なるタイプのデータを異なるディレクトリにマウント可能
  - 権限制御：読み取り専用マウントを設定して誤った変更を防止可能

- トレードオフ
  - 一つのSecret値 = 一つのファイルがベストプラクティス、異なる方法も可能だが相応の代償がある
  - Kubernetes specで複雑なコマンドを書くことを避ける
  - 設定の簡潔性と保守性を維持
  - アプリケーションに設定の組み合わせを処理させる

### 3. アプリケーション読み取り例

```python
# app.py
import os
import ssl
import requests

def load_secrets():
    """すべてのSecretを読み込み"""
    secrets = {}
    
    # API KEYを読み取り
    try:
        with open('/app/secrets/api-key.txt', 'r') as f:
            secrets['api_key'] = f.read().strip()
        print("API KEY読み込み成功")
    except FileNotFoundError:
        print("API KEYファイルが見つかりません")
    
    # データベースパスワードを読み取り
    try:
        with open('/app/config/db-password.txt', 'r') as f:
            secrets['db_password'] = f.read().strip()
        print("データベースパスワード読み込み成功")
    except FileNotFoundError:
        print("データベースパスワードファイルが見つかりません")
    
    # TLS証明書をチェック
    cert_path = "/etc/ssl/certs/server.crt"
    key_path = "/etc/ssl/certs/server.key"
    
    if os.path.exists(cert_path) and os.path.exists(key_path):
        secrets['tls_cert'] = cert_path
        secrets['tls_key'] = key_path
        print("TLS証明書読み込み成功")
    else:
        print("TLS証明書ファイルが見つかりません")
    
    return secrets

def main():
    secrets = load_secrets()
    
    # API KEYを使用
    if 'api_key' in secrets:
        headers = {"Authorization": f"Bearer {secrets['api_key']}"}
        print(f"API KEY使用: {secrets['api_key'][:10]}...")
    
    # データベースパスワードを使用
    if 'db_password' in secrets:
        print("読み込まれたデータベースパスワードを使用")
    
    # TLS証明書を使用
    if 'tls_cert' in secrets:
        print(f"TLS証明書パス: {secrets['tls_cert']}")

if __name__ == "__main__":
    main()
```

### 4. デプロイコマンド

```bash
# Secretを作成
kubectl apply -f secret.yaml

# Deploymentを作成
kubectl apply -f deployment.yaml

# Podステータスをチェック
kubectl get pods -l app=web-app

# Secretマウントを検証
kubectl exec -it <pod-name> -- ls -la /etc/ssl/certs/
kubectl exec -it <pod-name> -- ls -la /app/secrets/
kubectl exec -it <pod-name> -- ls -la /app/config/

# アプリケーションをテスト
kubectl exec -it <pod-name> -- python app.py
```

### 5. ファイル構造

```
Pod内部ファイル構造：
/etc/ssl/certs/
├── server.crt    # TLS証明書
└── server.key    # TLS秘密鍵

/app/secrets/
└── api-key.txt   # APIキー

/app/config/
└── db-password.txt  # データベースパスワード
```

### 6. セキュリティ考慮事項

1. **ファイル権限**：Secretファイルはデフォルトでrootのみ読み取り可能
2. **読み取り専用マウント**：`readOnly: true`を使用して変更を防止
3. **機密データ**：ログで完全なSecretを出力しないよう注意
4. **ファイルパス**：混乱を避けるため絶対パスを使用
