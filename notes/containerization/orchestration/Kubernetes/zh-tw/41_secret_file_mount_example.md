# Secret 檔案掛載實際範例

[English](../en/41_secret_file_mount_example.md) | [繁體中文](../zh-tw/41_secret_file_mount_example.md) | [日本語](../ja/41_secret_file_mount_example.md) | [回到索引](../README.md)

## 完整範例：TLS 憑證和 API KEY 管理

### 1. 建立 Secret

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

### 2. 建立 Deployment

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
          mountPath: /etc/ssl/certs # 把 tls-certs volume 掛載到 container 的 /etc/ssl/certs 目錄
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
          - key: tls.crt       # Secret 中的 key
            path: server.crt   # 在 volume 中要建立的檔案名稱
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
- 關鍵
  - 一對一映射
  - 每個 key 對應一個 path
  - key 是 Secret 中的欄位名稱
  - path 是 volume 中的檔案名稱

- 檔案重新命名
  - Secret 中的 tls.crt → volume 中的 server.crt
  - Secret 中的 tls.key → volume 中的 server.key
  - 這樣可以讓檔案名稱更符合應用程式的需求

- 多個 Volume 使用同一個 Secret
  - 同一個 Secret 可以被多個 volume 使用
  - 每個 volume 可以選擇不同的 key 和 path
  - 這樣可以將不同類型的資料掛載到不同的目錄

- 靈活的檔案組織
  - TLS 憑證掛載到 /etc/ssl/certs/ (標準 SSL 目錄)
  - API KEY 掛載到 /app/secrets/ (應用程式專用目錄)
  - 資料庫密碼掛載到 /app/config/ (配置目錄)

- 實際應用優勢
  - 安全性：敏感資料不會出現在環境變數中
  - 靈活性：可以重新命名檔案，符合應用程式需求
  - 組織性：不同類型的資料可以掛載到不同目錄
  - 權限控制：可以設定只讀掛載，防止意外修改

- 取捨
  - 一個 Secret 值 = 一個檔案 是最佳實踐，可以不這麼做，但會有相對應的代價
  - 避免在 Kubernetes spec 中寫複雜指令
  - 保持配置的簡潔性和可維護性
  - 讓應用程式負責處理配置的組合

### 3. 應用程式讀取範例

```python
# app.py
import os
import ssl
import requests

def load_secrets():
    """載入所有 Secret"""
    secrets = {}
    
    # 讀取 API KEY
    try:
        with open('/app/secrets/api-key.txt', 'r') as f:
            secrets['api_key'] = f.read().strip()
        print("API KEY 載入成功")
    except FileNotFoundError:
        print("API KEY 檔案不存在")
    
    # 讀取資料庫密碼
    try:
        with open('/app/config/db-password.txt', 'r') as f:
            secrets['db_password'] = f.read().strip()
        print("資料庫密碼載入成功")
    except FileNotFoundError:
        print("資料庫密碼檔案不存在")
    
    # 檢查 TLS 憑證
    cert_path = "/etc/ssl/certs/server.crt"
    key_path = "/etc/ssl/certs/server.key"
    
    if os.path.exists(cert_path) and os.path.exists(key_path):
        secrets['tls_cert'] = cert_path
        secrets['tls_key'] = key_path
        print("TLS 憑證載入成功")
    else:
        print("TLS 憑證檔案不存在")
    
    return secrets

def main():
    secrets = load_secrets()
    
    # 使用 API KEY
    if 'api_key' in secrets:
        headers = {"Authorization": f"Bearer {secrets['api_key']}"}
        print(f"使用 API KEY: {secrets['api_key'][:10]}...")
    
    # 使用資料庫密碼
    if 'db_password' in secrets:
        print("使用載入的資料庫密碼")
    
    # 使用 TLS 憑證
    if 'tls_cert' in secrets:
        print(f"TLS 憑證路徑: {secrets['tls_cert']}")

if __name__ == "__main__":
    main()
```

### 4. 部署指令

```bash
# 建立 Secret
kubectl apply -f secret.yaml

# 建立 Deployment
kubectl apply -f deployment.yaml

# 檢查 Pod 狀態
kubectl get pods -l app=web-app

# 驗證 Secret 掛載
kubectl exec -it <pod-name> -- ls -la /etc/ssl/certs/
kubectl exec -it <pod-name> -- ls -la /app/secrets/
kubectl exec -it <pod-name> -- ls -la /app/config/

# 測試應用程式
kubectl exec -it <pod-name> -- python app.py
```

### 5. 檔案結構

```
Pod 內部檔案結構：
/etc/ssl/certs/
├── server.crt    # TLS 憑證
└── server.key    # TLS 私鑰

/app/secrets/
└── api-key.txt   # API 金鑰

/app/config/
└── db-password.txt  # 資料庫密碼
```

### 6. 安全注意事項

1. **檔案權限**：Secret 檔案預設只有 root 可讀
2. **只讀掛載**：使用 `readOnly: true` 防止修改
3. **敏感資料**：避免在日誌中輸出完整 Secret
4. **檔案路徑**：使用絕對路徑避免混淆
