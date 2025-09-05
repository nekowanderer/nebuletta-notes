# ConfigMap と Secret

[English](../en/40_configmap_and_secret.md) | [繁體中文](../zh-tw/40_configmap_and_secret.md) | [日本語](../ja/40_configmap_and_secret.md) | [回到索引](../README.md)

## 概要

ConfigMap と Secret は、アプリケーションの設定と機密データを管理するための Kubernetes の2つのリソースです。これらにより、設定をアプリケーションコードから分離し、柔軟な設定管理とセキュリティを実現できます。

## 1. ConfigMap - 設定管理

### 1.1 基本概念

ConfigMap は、環境変数、設定ファイルなどの非機密な設定データを保存するために使用されます。

### 1.2 ConfigMap の作成

```yaml
# hellok8s-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: hellok8s-config
data:
  MESSAGE: "It works with a ConfigMap!"
```

### 1.3 Pod での ConfigMap の使用

#### 方法1: 単一環境変数参照

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

#### 方法2: 全設定の一括インポート (envFrom)

複数の環境変数がある場合、`envFrom` を使用してすべての設定を一度にインポートできます：

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

**注意**: `envFrom` を使用する場合、環境変数名は ConfigMap で定義された key 名を直接使用します。

#### 方法3: 変数名の競合回避 (プレフィックス使用)

複数の ConfigMap から設定をインポートする場合、変数名の競合が発生する可能性があります。プレフィックスを使用してこれを回避できます：

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

これにより、`hellok8s-config` からインポートされたすべての変数に `CONFIG_` プレフィックスが追加されます。

### 1.4 ConfigMap をファイルとしてマウント

アプリケーションが設定ファイルを読み取る必要がある場合、ConfigMap を Volume としてマウントできます：

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

## 2. Secret - 機密データ管理

### 2.1 基本概念

Secret は、パスワード、API キー、TLS 証明書などの機密データを保存するために使用されます。Kubernetes は Secret 内のデータを base64 エンコードします。

### 2.2 Secret の作成

#### 方法1: data フィールドの使用 (base64 エンコードが必要)

```bash
# まず機密データを base64 エンコード
echo 'It works with a Secret' | base64
# 出力: SXQgd29ya3Mgd2l0aCBhIFNlY3JldAo=
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

#### 方法2: stringData フィールドの使用 (自動エンコード)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: hellok8s-secret
stringData:
  SECRET_MESSAGE: "It works with a Secret"
```

### 2.3 Pod での Secret の使用

#### 方法1: 単一環境変数参照

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

#### 方法2: 全 Secret の一括インポート

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

### 2.4 Secret をファイルとしてマウント

#### 基本例

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

## 3. 環境変数 vs ファイルマウント

### 3.1 環境変数の利点と欠点

**利点**:
- シンプルで直接的な設定
- アプリケーションが読み取りやすい
- 小さな設定に適している

**欠点**:
- 機密データが誤って漏洩する可能性がある（アプリケーションがクラッシュ時にすべての環境変数をダンプする）
- 子プロセスが環境変数を継承する
- ログシステムが機密情報を記録する可能性がある
- 大きな設定ファイルには適していない

### 3.2 ファイルマウントの利点と欠点

**利点**:
- より安全、ファイル権限を制御できる
- 大きな設定ファイルに適している
- 機密データが環境変数に表示されない
- 複雑な設定ファイル形式をサポート

**欠点**:
- 設定が比較的複雑
- アプリケーションがファイル読み取りをサポートする必要がある

### 3.3 ベストプラクティスの推奨事項

1. **一般的な設定**: ConfigMap + 環境変数を使用
2. **機密データ**: Secret + ファイルマウントを使用
3. **大きな設定ファイル**: ConfigMap + ファイルマウントを使用
4. **複数の設定ソース**: プレフィックスを使用して変数名の競合を回避
5. **TLS 証明書**: Secret + ファイルマウントを使用

## 4. まとめ

ConfigMap と Secret は Kubernetes の設定管理の中核ツールです：

- **ConfigMap**: 非機密な設定データを管理
- **Secret**: 機密データを管理
- **環境変数**: 小さな、非機密な設定に適している
- **ファイルマウント**: 大きな設定ファイルや機密データに適している

環境変数とファイルマウントのどちらを使用するかは、データの機密性、サイズ、アプリケーションの要件に基づいて決定する必要があります。
