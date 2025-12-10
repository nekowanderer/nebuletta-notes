# Kubernetes Dashboard RBAC 權限檢查與範例

[English](../en/43_k8s_rbac.md) | [繁體中文](../zh-tw/43_k8s_rbac.md) | [日本語](../ja/43_k8s_rbac.md) | [回到索引](../README.md)

前一節安裝 Kubernetes Dashboard 的時候為了登入所以有建立一個 admin user，這節簡單說明這部分的概念

## 檢查 admin-user 權限

建立 account 的 yaml:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

### 1. 看 ClusterRoleBinding 是否綁對
```bash
$ kubectl get clusterrolebinding admin-user -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"rbac.authorization.k8s.io/v1","kind":"ClusterRoleBinding","metadata":{"annotations":{},"name":"admin-user"},"roleRef":{"apiGroup":"rbac.authorization.k8s.io","kind":"ClusterRole","name":"cluster-admin"},"subjects":[{"kind":"ServiceAccount","name":"admin-user","namespace":"kubernetes-dashboard"}]}
  creationTimestamp: "2025-09-12T14:11:00Z"
  name: admin-user
  resourceVersion: "1666117"
  uid: b95fa22a-f73c-4fae-9040-043915feb6fd
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

重點要看到：
```yaml
roleRef:
  kind: ClusterRole
  name: cluster-admin   # ⬅️ 這就是授予 admin 權限的關鍵
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

---

### 2. 看 cluster-admin 角色本身定義
```bash
$ kubectl get clusterrole cluster-admin -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: "2025-01-19T05:40:14Z"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: cluster-admin
  resourceVersion: "78"
  uid: 0a78b70c-71e4-4cfd-8744-ce717844e759
rules:
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs:
  - '*'
- nonResourceURLs:
  - '*'
  verbs:
  - '*'
```

會看到：
```yaml
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```
➡️ 代表對所有 API 群組、所有資源、所有操作都有權限。

---

### 3. 模擬 ServiceAccount 權限 (單一動作)
```bash
$ kubectl auth can-i list pods --as=system:serviceaccount:kubernetes-dashboard:admin-user
yes
```

---

### 4. 列出 ServiceAccount 所有權限
```bash
$ kubectl auth can-i --list --as=system:serviceaccount:kubernetes-dashboard:admin-user
Resources                                       Non-Resource URLs                      Resource Names   Verbs
*.*                                             []                                     []               [*]
                                                [*]                                    []               [*]
selfsubjectreviews.authentication.k8s.io        []                                     []               [create]
selfsubjectaccessreviews.authorization.k8s.io   []                                     []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                                     []               [create]
                                                [/.well-known/openid-configuration/]   []               [get]
                                                [/.well-known/openid-configuration]    []               [get]
                                                [/api/*]                               []               [get]
                                                [/api]                                 []               [get]
                                                [/apis/*]                              []               [get]
                                                [/apis]                                []               [get]
                                                [/healthz]                             []               [get]
                                                [/healthz]                             []               [get]
                                                [/livez]                               []               [get]
                                                [/livez]                               []               [get]
                                                [/openapi/*]                           []               [get]
                                                [/openapi]                             []               [get]
                                                [/openid/v1/jwks/]                     []               [get]
                                                [/openid/v1/jwks]                      []               [get]
                                                [/readyz]                              []               [get]
                                                [/readyz]                              []               [get]
                                                [/version/]                            []               [get]
                                                [/version/]                            []               [get]
                                                [/version]                             []               [get]
                                                [/version]                             []               [get]
```

幾乎所有資源應該都會是 `yes`。  

---

## 建立一個「Read-Only Dashboard User」

有時候不想給 cluster-admin，那可以只給 **讀取權限**。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-read-only
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: dashboard-read-only
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets", "namespaces", "nodes"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets", "daemonsets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["metrics.k8s.io"]
  resources: ["pods", "nodes"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dashboard-read-only
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: dashboard-read-only
subjects:
- kind: ServiceAccount
  name: dashboard-read-only
  namespace: kubernetes-dashboard
```

---

### 用法
1. 建立資源：
   ```bash
   kubectl apply -f dashboard-read-only.yaml
   ```
2. 產生 token：
   ```bash
   kubectl -n kubernetes-dashboard create token dashboard-read-only --duration=24h
   ```
3. 用這個 token 登入 Dashboard，就只能瀏覽，不能修改 / 刪除。

---

## 總結
- `roleRef.name: cluster-admin` → 就是讓 `admin-user` 擁有最高權限的關鍵。  
- 用 `kubectl auth can-i` 可以模擬權限。  
- 想要安全一點，可以給「Read-Only User」來瀏覽 Dashboard，不要一律用 admin。
