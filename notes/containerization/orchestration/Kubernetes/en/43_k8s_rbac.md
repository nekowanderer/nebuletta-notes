# Kubernetes Dashboard RBAC Permission Check and Examples

[English](../en/43_k8s_rbac.md) | [繁體中文](../zh-tw/43_k8s_rbac.md) | [日本語](../ja/43_k8s_rbac.md) | [Back to Index](../README.md)

In the previous section, we created an admin user to log in to the Kubernetes Dashboard. This section briefly explains the concepts behind this setup.

## Check admin-user Permissions

Create the account yaml:
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

### 1. Check if ClusterRoleBinding is correctly bound
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

Key points to check:
```yaml
roleRef:
  kind: ClusterRole
  name: cluster-admin   # ⬅️ This is the key that grants admin permissions
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

---

### 2. Check the cluster-admin role definition
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

You will see:
```yaml
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```
➡️ This means permissions for all API groups, all resources, and all operations.

---

### 3. Simulate ServiceAccount permissions (single action)
```bash
$ kubectl auth can-i list pods --as=system:serviceaccount:kubernetes-dashboard:admin-user
yes
```

---

### 4. List all ServiceAccount permissions
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

Almost all resources should return `yes`.

---

## Create a "Read-Only Dashboard User"

Sometimes you don't want to grant cluster-admin permissions. In such cases, you can grant **read-only permissions** instead.

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

### Usage
1. Create resources:
   ```bash
   kubectl apply -f dashboard-read-only.yaml
   ```
2. Generate token:
   ```bash
   kubectl -n kubernetes-dashboard create token dashboard-read-only --duration=24h
   ```
3. Use this token to log in to the Dashboard. You will only be able to browse, not modify or delete.

---

## Summary
- `roleRef.name: cluster-admin` → This is the key that grants `admin-user` the highest permissions.
- Use `kubectl auth can-i` to simulate permissions.
- For better security, you can create a "Read-Only User" to browse the Dashboard instead of always using admin.

