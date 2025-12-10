# Kubernetes Dashboard RBAC 権限チェックと例

[English](../en/43_k8s_rbac.md) | [繁體中文](../zh-tw/43_k8s_rbac.md) | [日本語](../ja/43_k8s_rbac.md) | [インデックスに戻る](../README.md)

前節で Kubernetes Dashboard をインストールする際、ログイン用に admin user を作成しました。本節では、この部分の概念について簡単に説明します。

## admin-user の権限を確認する

アカウントの yaml を作成：
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

### 1. ClusterRoleBinding が正しくバインドされているか確認
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

確認すべきポイント：
```yaml
roleRef:
  kind: ClusterRole
  name: cluster-admin   # ⬅️ これが admin 権限を付与するキー
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

---

### 2. cluster-admin ロール自体の定義を確認
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

以下のように表示されます：
```yaml
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```
➡️ すべての API グループ、すべてのリソース、すべての操作に対する権限を意味します。

---

### 3. ServiceAccount の権限をシミュレート（単一アクション）
```bash
$ kubectl auth can-i list pods --as=system:serviceaccount:kubernetes-dashboard:admin-user
yes
```

---

### 4. ServiceAccount のすべての権限を一覧表示
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

ほぼすべてのリソースが `yes` になるはずです。

---

## 「Read-Only Dashboard User」を作成する

cluster-admin を付与したくない場合があります。そのような場合は、**読み取り専用権限**のみを付与できます。

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

### 使用方法
1. リソースを作成：
   ```bash
   kubectl apply -f dashboard-read-only.yaml
   ```
2. トークンを生成：
   ```bash
   kubectl -n kubernetes-dashboard create token dashboard-read-only --duration=24h
   ```
3. このトークンを使用して Dashboard にログインすると、閲覧のみ可能で、変更や削除はできません。

---

## まとめ
- `roleRef.name: cluster-admin` → これが `admin-user` に最高権限を付与するキーです。
- `kubectl auth can-i` を使用して権限をシミュレートできます。
- セキュリティを向上させるため、常に admin を使用するのではなく、「Read-Only User」を作成して Dashboard を閲覧できます。

