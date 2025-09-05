# ConfigMap 與 Secret

[English](../en/40_configmap_and_secret.md) | [繁體中文](../zh-tw/40_configmap_and_secret.md) | [日本語](../ja/40_configmap_and_secret.md) | [回到索引](../README.md)

## 概述

ConfigMap 和 Secret 是 Kubernetes 中用來管理應用程式配置和敏感資料的兩種資源。它們讓我們可以將配置從應用程式程式碼中分離出來，實現配置的靈活管理和安全性。

## 1. ConfigMap - 配置管理

### 1.1 基本概念

ConfigMap 用來儲存非敏感的配置資料，如環境變數、配置檔案等。

### 1.2 建立 ConfigMap

```yaml
# hellok8s-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: hellok8s-config
data:
  MESSAGE: "It works with a ConfigMap!"
```

### 1.3 在 Pod 中使用 ConfigMap

#### 方法一：單一環境變數引用

```yaml
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
      - image: brianstorti/hellok8s:v4
        name: hellok8s-container
        env:
          - name: MESSAGE
            valueFrom:
              configMapKeyRef:
                name: hellok8s-config
                key: MESSAGE
```

#### 方法二：批量導入所有配置 (envFrom)

當有多個環境變數時，使用 `envFrom` 可以一次導入所有配置：

```yaml
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
      - image: brianstorti/hellok8s:v4
        name: hellok8s-container
        envFrom:
          - configMapRef:
              name: hellok8s-config
```

**注意**：使用 `envFrom` 時，環境變數名稱會直接使用 ConfigMap 中定義的 key 名稱。

#### 方法三：避免變數名稱衝突 (使用前綴)

當從多個 ConfigMap 導入配置時，可能會發生變數名稱衝突。可以使用前綴來避免：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hellok8s
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hellok8s
  template:
    metadata:
      labels:
        app: hellok8s
    spec:
      containers:
      - image: brianstorti/hellok8s:v4
        name: hellok8s-container
        envFrom:
        - configMapRef:
            name: hellok8s-config
          prefix: CONFIG_
```

這樣所有從 `hellok8s-config` 導入的變數都會加上 `CONFIG_` 前綴。

### 1.4 將 ConfigMap 掛載為檔案

當應用程式需要讀取配置檔案時，可以將 ConfigMap 掛載為 Volume：

```yaml
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
      volumes:
       - name: config
         configMap:
           name: hellok8s-config
      containers:
      - image: brianstorti/hellok8s:v4
        name: hellok8s-container
        volumeMounts:
        - name: config
          mountPath: /config
```

## 2. Secret - 敏感資料管理

### 2.1 基本概念

Secret 用來儲存敏感的資料，如密碼、API 金鑰、TLS 憑證等。Kubernetes 會對 Secret 中的資料進行 base64 編碼。

### 2.2 建立 Secret

#### 方法一：使用 data 欄位（需要 base64 編碼）

```bash
# 先將敏感資料進行 base64 編碼
echo 'It works with a Secret' | base64
# 輸出：SXQgd29ya3Mgd2l0aCBhIFNlY3JldAo=
```

```yaml
# hellok8s-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: hellok8s-secret
data:
  SECRET_MESSAGE: "SXQgd29ya3Mgd2l0aCBhIFNlY3JldAo="
```

#### 方法二：使用 stringData 欄位（自動編碼）

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: hellok8s-secret
stringData:
  SECRET_MESSAGE: "It works with a Secret"
```

### 2.3 在 Pod 中使用 Secret

#### 方法一：單一環境變數引用

```yaml
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
      - image: brianstorti/hellok8s:v4
        name: hellok8s-container
        env:
          - name: MESSAGE
            valueFrom:
              secretKeyRef:
                name: hellok8s-secret
                key: SECRET_MESSAGE
```

#### 方法二：批量導入所有 Secret

```yaml
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
      - image: brianstorti/hellok8s:v4
        name: hellok8s-container
        envFrom:
         - secretRef:
             name: hellok8s-secret
```

### 2.4 將 Secret 掛載為檔案

#### 基本範例

```yaml
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
      volumes:
        - name: secrets
          secret:
            secretName: hellok8s-secret
      containers:
      - image: brianstorti/hellok8s:v4
        name: hellok8s-container
        volumeMounts:
          - name: secrets
            mountPath: /secrets
```

## 3. 環境變數 vs 檔案掛載的選擇

### 3.1 環境變數的優缺點

**優點**：
- 設定簡單直接
- 應用程式容易讀取
- 適合小型配置

**缺點**：
- 敏感資料可能被意外洩露（應用程式崩潰時會輸出所有環境變數）
- 子程序會繼承環境變數
- 日誌系統可能記錄敏感資訊
- 不適合大型配置檔案

### 3.2 檔案掛載的優缺點

**優點**：
- 更安全，可以控制檔案權限
- 適合大型配置檔案
- 敏感資料不會出現在環境變數中
- 支援複雜的配置檔案格式

**缺點**：
- 設定相對複雜
- 需要應用程式支援檔案讀取

### 3.3 最佳實踐建議

1. **一般配置**：使用 ConfigMap + 環境變數
2. **敏感資料**：使用 Secret + 檔案掛載
3. **大型配置檔案**：使用 ConfigMap + 檔案掛載
4. **多個配置來源**：使用前綴避免變數名稱衝突
5. **TLS 憑證**：使用 Secret + 檔案掛載

## 4. 總結

ConfigMap 和 Secret 是 Kubernetes 配置管理的核心工具：

- **ConfigMap**：管理非敏感的配置資料
- **Secret**：管理敏感的資料
- **環境變數**：適合小型、非敏感的配置
- **檔案掛載**：適合大型配置檔案或敏感資料

選擇使用環境變數還是檔案掛載，需要根據資料的敏感性、大小和應用程式的需求來決定。
