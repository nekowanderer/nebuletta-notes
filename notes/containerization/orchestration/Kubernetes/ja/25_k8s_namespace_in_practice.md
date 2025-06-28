# Kubernetes Namespace 実践操作

[English](../en/25_k8s_namespace_in_practice.md) | [繁體中文](../zh-tw/25_k8s_namespace_in_practice.md) | [日本語](../ja/25_k8s_namespace_in_practice.md) | [インデックスに戻る](../README.md)

## 前提条件

### Docker環境の起動
```bash
# Dockerサービスの開始
$ sudo service docker start

# Dockerの状態確認
$ docker ps
```

### Minikubeクラスターの構築
```bash
# DockerドライバーでMinikubeを起動
$ minikube start --driver docker

# クラスターの状態確認
$ minikube status
```

## NamespaceとアプリケーションGの作成

### YAML設定ファイルの作成
`beta-app-all.yaml`ファイルを作成します：

```yaml
# namespaceの作成
apiVersion: v1
kind: Namespace
metadata:
  name: app-ns
---
# Deploymentの作成
apiVersion: apps/v1
kind: Deployment
metadata:
  name: beta-app-deployment
  namespace: app-ns
spec:
  replicas: 3
  selector:
    matchLabels:
      app: beta-app-pod
  template:
    metadata:
      labels:
        app: beta-app-pod
    spec:
      containers:
      - name: beta-app-container
        image: uopsdod/k8s-hostname-amd64-beta:v1
        ports: 
        - containerPort: 80
---
# Serviceの作成
apiVersion: v1
kind: Service
metadata:
  name: beta-app-service-clusterip
  namespace: app-ns
spec:
  selector:
    app: beta-app-pod
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
```
ここで、`---`は同一のyamlファイル内で複数のリソースタイプをデプロイすることを表します。

## デプロイと検証

### アプリケーションのデプロイ
```bash
# 全リソースのデプロイ
$ kubectl apply -f beta-app-all.yaml
```

### デプロイ状態の確認
```bash
# 全namespaceの表示
$ kubectl get ns
NAME              STATUS   AGE
app-ns            Active   7s
default           Active   7d9h
kube-node-lease   Active   7d9h
kube-public       Active   7d9h
kube-system       Active   7d9h

# 全リソースの表示（デフォルトnamespace）
$ kubectl get all
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   5d

# 指定したnamespace内の全リソースの表示
$ kubectl get all --namespace app-ns
NAME                                       READY   STATUS    RESTARTS   AGE
pod/beta-app-deployment-865c646d9d-9ml47   1/1     Running   0          54s
pod/beta-app-deployment-865c646d9d-shc9b   1/1     Running   0          54s
pod/beta-app-deployment-865c646d9d-xl8v6   1/1     Running   0          54s

NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/beta-app-service-clusterip   ClusterIP   10.103.190.75   <none>        8080/TCP   54s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/beta-app-deployment   3/3     3            3           54s

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/beta-app-deployment-865c646d9d   3         3         3       54s

# または短縮形を使用
$ kubectl get all -n app-ns
```

## リソースのクリーンアップ

### Namespaceの削除
```bash
# namespace全体を削除（そのnamespace内の全リソースも同時に削除される）
$ kubectl delete ns app-ns
namespace "app-ns" deleted

# namespaceが削除されたことを確認
$ kubectl get ns
NAME              STATUS   AGE
default           Active   7d9h
kube-node-lease   Active   7d9h
kube-public       Active   7d9h
kube-system       Active   7d9h

# 削除されたnamespace内のリソースを表示してみる（エラーが表示されるはず）
$ kubectl get all -n app-ns
No resources found in app-ns namespace.
```

## 重要な概念の説明

- **Namespace分離**：異なるnamespace内のリソースは相互に分離されています
- **リソースの所属**：DeploymentとServiceは共に指定されたnamespaceに属します
- **削除の動作**：namespaceを削除すると、そのnamespace内の全リソースが同時に削除されます
- **Namespace間アクセス**：デフォルトでは、異なるnamespaceのリソースは直接アクセスできません
