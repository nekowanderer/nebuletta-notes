# Secret File Mounting Practical Example

[English](../en/41_secret_file_mount_example.md) | [繁體中文](../zh-tw/41_secret_file_mount_example.md) | [日本語](../ja/41_secret_file_mount_example.md) | [回到索引](../README.md)

## Complete Example: TLS Certificate and API KEY Management

### 1. Creating Secret

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

### 2. Creating Deployment

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
          mountPath: /etc/ssl/certs # Mount tls-certs volume to container's /etc/ssl/certs directory
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
          - key: tls.crt       # Key in Secret
            path: server.crt   # File name to create in volume
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

- Key Points
  - One-to-one mapping
  - Each key corresponds to one path
  - key is the field name in Secret
  - path is the file name in volume

- File renaming
  - tls.crt in Secret → server.crt in volume
  - tls.key in Secret → server.key in volume
  - This allows file names to better match application requirements

- Multiple volumes using the same Secret
  - The same Secret can be used by multiple volumes
  - Each volume can select different keys and paths
  - This allows different types of data to be mounted to different directories

- Flexible file organization
  - TLS certificates mounted to /etc/ssl/certs/ (standard SSL directory)
  - API KEY mounted to /app/secrets/ (application-specific directory)
  - Database password mounted to /app/config/ (configuration directory)

- Practical application advantages
  - Security: Sensitive data won't appear in environment variables
  - Flexibility: Can rename files to match application requirements
  - Organization: Different types of data can be mounted to different directories
  - Permission control: Can set read-only mounting to prevent accidental modifications

- Trade-offs
  - One Secret value = one file is the best practice, can be done differently but with corresponding costs
  - Avoid writing complex commands in Kubernetes specs
  - Maintain configuration simplicity and maintainability
  - Let applications handle configuration combination

### 3. Application Reading Example

```python
# app.py
import os
import ssl
import requests

def load_secrets():
    """Load all Secrets"""
    secrets = {}
    
    # Read API KEY
    try:
        with open('/app/secrets/api-key.txt', 'r') as f:
            secrets['api_key'] = f.read().strip()
        print("API KEY loaded successfully")
    except FileNotFoundError:
        print("API KEY file not found")
    
    # Read database password
    try:
        with open('/app/config/db-password.txt', 'r') as f:
            secrets['db_password'] = f.read().strip()
        print("Database password loaded successfully")
    except FileNotFoundError:
        print("Database password file not found")
    
    # Check TLS certificates
    cert_path = "/etc/ssl/certs/server.crt"
    key_path = "/etc/ssl/certs/server.key"
    
    if os.path.exists(cert_path) and os.path.exists(key_path):
        secrets['tls_cert'] = cert_path
        secrets['tls_key'] = key_path
        print("TLS certificates loaded successfully")
    else:
        print("TLS certificate files not found")
    
    return secrets

def main():
    secrets = load_secrets()
    
    # Use API KEY
    if 'api_key' in secrets:
        headers = {"Authorization": f"Bearer {secrets['api_key']}"}
        print(f"Using API KEY: {secrets['api_key'][:10]}...")
    
    # Use database password
    if 'db_password' in secrets:
        print("Using loaded database password")
    
    # Use TLS certificates
    if 'tls_cert' in secrets:
        print(f"TLS certificate path: {secrets['tls_cert']}")

if __name__ == "__main__":
    main()
```

### 4. Deployment Commands

```bash
# Create Secret
kubectl apply -f secret.yaml

# Create Deployment
kubectl apply -f deployment.yaml

# Check Pod status
kubectl get pods -l app=web-app

# Verify Secret mounting
kubectl exec -it <pod-name> -- ls -la /etc/ssl/certs/
kubectl exec -it <pod-name> -- ls -la /app/secrets/
kubectl exec -it <pod-name> -- ls -la /app/config/

# Test application
kubectl exec -it <pod-name> -- python app.py
```

### 5. File Structure

```
Pod internal file structure:
/etc/ssl/certs/
├── server.crt    # TLS certificate
└── server.key    # TLS private key

/app/secrets/
└── api-key.txt   # API key

/app/config/
└── db-password.txt  # Database password
```

### 6. Security Considerations

1. **File permissions**: Secret files are readable only by root by default
2. **Read-only mounting**: Use `readOnly: true` to prevent modifications
3. **Sensitive data**: Avoid outputting complete Secret in logs
4. **File paths**: Use absolute paths to avoid confusion
