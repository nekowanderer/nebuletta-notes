# ConfigMap and Secret

[English](../en/40_configmap_and_secret.md) | [繁體中文](../zh-tw/40_configmap_and_secret.md) | [日本語](../ja/40_configmap_and_secret.md) | [回到索引](../README.md)

## Overview

ConfigMap and Secret are two Kubernetes resources used to manage application configuration and sensitive data. They allow us to separate configuration from application code, enabling flexible configuration management and security.

## 1. ConfigMap - Configuration Management

### 1.1 Basic Concepts

ConfigMap is used to store non-sensitive configuration data, such as environment variables, configuration files, etc.

### 1.2 Creating ConfigMap

```yaml
# hellok8s-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: hellok8s-config
data:
  MESSAGE: "It works with a ConfigMap!"
```

### 1.3 Using ConfigMap in Pods

#### Method 1: Single Environment Variable Reference

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

#### Method 2: Bulk Import All Configurations (envFrom)

When there are multiple environment variables, using `envFrom` can import all configurations at once:

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

**Note**: When using `envFrom`, the environment variable names will directly use the key names defined in the ConfigMap.

#### Method 3: Avoiding Variable Name Conflicts (Using Prefix)

When importing configurations from multiple ConfigMaps, variable name conflicts may occur. Use prefixes to avoid this:

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

This way, all variables imported from `hellok8s-config` will have the `CONFIG_` prefix added.

### 1.4 Mounting ConfigMap as Files

When applications need to read configuration files, ConfigMap can be mounted as a Volume:

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

## 2. Secret - Sensitive Data Management

### 2.1 Basic Concepts

Secret is used to store sensitive data, such as passwords, API keys, TLS certificates, etc. Kubernetes will base64 encode the data in Secrets.

### 2.2 Creating Secret

#### Method 1: Using data field (requires base64 encoding)

```bash
# First encode sensitive data with base64
echo 'It works with a Secret' | base64
# Output: SXQgd29ya3Mgd2l0aCBhIFNlY3JldAo=
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

#### Method 2: Using stringData field (auto-encoding)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: hellok8s-secret
stringData:
  SECRET_MESSAGE: "It works with a Secret"
```

### 2.3 Using Secret in Pods

#### Method 1: Single Environment Variable Reference

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

#### Method 2: Bulk Import All Secrets

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

### 2.4 Mounting Secret as Files

#### Basic Example

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

## 3. Environment Variables vs File Mounting

### 3.1 Pros and Cons of Environment Variables

**Pros**:
- Simple and direct setup
- Easy for applications to read
- Suitable for small configurations

**Cons**:
- Sensitive data may be accidentally leaked (applications dump all environment variables when they crash)
- Child processes inherit environment variables
- Logging systems may record sensitive information
- Not suitable for large configuration files

### 3.2 Pros and Cons of File Mounting

**Pros**:
- More secure, can control file permissions
- Suitable for large configuration files
- Sensitive data won't appear in environment variables
- Supports complex configuration file formats

**Cons**:
- Relatively complex setup
- Requires application support for file reading

### 3.3 Best Practice Recommendations

1. **General Configuration**: Use ConfigMap + Environment Variables
2. **Sensitive Data**: Use Secret + File Mounting
3. **Large Configuration Files**: Use ConfigMap + File Mounting
4. **Multiple Configuration Sources**: Use prefixes to avoid variable name conflicts
5. **TLS Certificates**: Use Secret + File Mounting

## 4. Summary

ConfigMap and Secret are core tools for Kubernetes configuration management:

- **ConfigMap**: Manages non-sensitive configuration data
- **Secret**: Manages sensitive data
- **Environment Variables**: Suitable for small, non-sensitive configurations
- **File Mounting**: Suitable for large configuration files or sensitive data

The choice between environment variables and file mounting depends on the sensitivity of the data, size, and application requirements.
