# 使用 Helm 安裝 Ingress Controller 的經驗分享

[English](../en/09_install_ingress_controller.md) | [繁體中文](09_install_ingress_controller.md) | [日本語](../ja/09_install_ingress_controller.md) | [返回索引](../README.md)

---

## 背景
- 實驗日期: 2025/08/28
- 難度：🤬
- 描述: 原本使用 `kubectl apply -f deploy.yaml` 安裝 ingress-nginx controller 時遇到版本過舊、設定複雜等問題，改用 Helm 安裝後大幅簡化流程。

## 重要概念澄清

在 Kubernetes 中，有兩個不同的概念需要區分：

- **Ingress Controller**：實際處理流量的程式（如 nginx），負責實現 Ingress 規則
- **Ingress**：定義路由規則的 Kubernetes 資源，告訴 Controller 如何轉發流量

Helm 安裝的是 **Ingress Controller**，然後我們需要另外建立 **Ingress** 資源來定義路由規則。

---

## 遇到的現象

- 使用官方提供的 `deploy.yaml` 檔案安裝 ingress-nginx controller
- 安裝後發現版本過舊，功能不完整，Pod 還會一直重啟
- 需要手動處理 Webhook、Certificate、RBAC 等複雜設定
- 每次更新都需要重新下載新的 YAML 檔案
- 移除時需要手動刪除多個資源

## 除錯過程

### 第一階段：嘗試傳統安裝方式

```bash
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.44.0/deploy/static/provider/cloud/deploy.yaml
```

**遇到的問題：**
- 版本固定，無法輕易升級
- YAML 檔案包含大量手動設定的資源
- 需要手動處理 namespace、service account、cluster role 等
- 移除時需要手動刪除所有相關資源

### 第二階段：改用 Helm 安裝

發現 Helm 可以解決上述所有問題，決定改用 Helm 安裝。

## 解決方案

### 步驟 1：使用 Helm 安裝 Ingress Controller

#### 1.1 加入 Helm Repository
```bash
$ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```

#### 1.2 更新 Repository
```bash
$ helm repo update
```

#### 1.3 安裝 Ingress Controller
```bash
$ helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

### 客製化安裝（可選）

如果需要使用 NodePort 而非預設的 LoadBalancer：

```bash
$ helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.http=30080 \
  --set controller.service.nodePorts.https=30443
```

### 步驟 2：建立資源

安裝完 Ingress Controller 後，我們需要建立 Ingress 資源來定義路由規則：

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

然後是 Service：

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
然後按順序 apply 即可


## 驗證安裝

### 檢查 Ingress Controller 狀態

#### 檢查 Pod 狀態
```bash
$ kubectl get pods -n ingress-nginx
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-66cdbcf9bf-g995j   1/1     Running   0          44m
```

#### 檢查 Service 狀態
```bash
$ kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP        EXTERNAL-IP     PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   192.168.194.148   192.168.139.2   80:31802/TCP,443:31809/TCP   44m
ingress-nginx-controller-admission   ClusterIP      192.168.194.149   <none>          443/TCP                      44m
```

### 檢查 Ingress 資源狀態
```bash
$ kubectl get ingress
NAME            CLASS   HOSTS   ADDRESS         PORTS   AGE
hello-ingress   nginx   *       192.168.139.2   80      42m
```

### 測試連線
```bash
curl http://192.168.139.2/hello
```

## 查看 Ingress Controller 版本

### 方法 1：使用 Helm 查看 Release 資訊
```bash
# 查看 release 的詳細資訊
$ helm list -n ingress-nginx
NAME         	NAMESPACE    	REVISION	UPDATED                             	STATUS  	CHART               	APP VERSION
ingress-nginx	ingress-nginx	1       	2025-08-28 21:13:57.830894 +0800 CST	deployed	ingress-nginx-4.13.1	1.13.1

# 查看 release 的詳細狀態
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

### 方法 2：查看 Chart 版本資訊
```bash
# 查看 chart 的 metadata
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

### 方法 3：查看 Pod 中的版本資訊
```bash
# 查看 ingress controller pod 的詳細資訊
$ kubectl describe pod -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# 或者直接查看 pod 的 image 標籤
$ kubectl get pods -n ingress-nginx -o jsonpath='{.items[0].spec.containers[0].image}'
registry.k8s.io/ingress-nginx/controller:v1.13.1@sha256:37e489b22ac77576576e52e474941cd7754238438847c1ee795ad6d59c02b12a%
```

### 方法 4：查看可用的 Chart 版本
```bash
# 查看遠端 repository 中可用的版本
$ helm search repo ingress-nginx/ingress-nginx --versions
NAME                       	CHART VERSION	APP VERSION	DESCRIPTION
ingress-nginx/ingress-nginx	4.13.1       	1.13.1     	Ingress controller for Kubernetes using NGINX a...
ingress-nginx/ingress-nginx	4.13.0       	1.13.0     	Ingress controller for Kubernetes using NGINX a...
ingress-nginx/ingress-nginx	4.12.5       	1.12.5     	Ingress controller for Kubernetes using NGINX a...
ingress-nginx/ingress-nginx	4.12.4       	1.12.4     	Ingress controller for Kubernetes using NGINX a...
...

# 查看 local chart 資訊
$ helm show chart ingress-nginx/ingress-nginx
```

## 為什麼 Helm 更好？

| 項目 | kubectl apply | Helm |
|------|---------------|------|
| 版本管理 | 需要手動下載新版本 | 可指定版本，自動處理 |
| 設定複雜度 | 需要手動處理所有資源 | 自動處理 Webhook、Certificate、RBAC |
| 客製化 | 需要修改 YAML | 使用 `--set` 參數 |
| 移除 | 需要手動刪除多個資源 | `helm uninstall` 一次完成 |
| 升級 | 需要重新下載並應用 | `helm upgrade` 自動處理 |

## 總結

原本使用 `kubectl apply -f deploy.yaml` 的方式雖然直接，但遇到版本管理、設定複雜度等問題。改用 Helm 後，整個安裝流程變得更加簡潔且易於維護，特別適合需要經常更新或客製化的環境。

## 重要提醒

**Helm 安裝的是 Ingress Controller，不是 Ingress 本身**。安裝完 Controller 後，還是需要：

1. 建立 Ingress 資源來定義路由規則
2. 確保 Ingress 資源中的 `ingressClassName` 與 Controller 匹配
3. 建立對應的 Service 和 Pod 來處理實際的業務邏輯

這樣的架構讓 Ingress Controller 和 Ingress 規則分離，提供了更好的靈活性和可維護性。
