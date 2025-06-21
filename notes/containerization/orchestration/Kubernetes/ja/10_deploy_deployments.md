# Kubernetes Deployment デプロイ実践

[English](../en/10_deploy_deployments.md) | [繁體中文](../zh-tw/10_deploy_deployments.md) | [日本語](../ja/10_deploy_deployments.md) | [インデックスに戻る](../README.md)

## 前提条件

#### Docker サービスの起動
```bash
$ sudo service docker start
$ docker ps
```

#### Minikube クラスターの作成
```bash
$ minikube start --driver docker
$ minikube status
```

## Deployment の作成

#### Deployment 定義ファイルの作成
`simple-deployment.yaml` ファイルを作成：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app-pod
  template:
    metadata:
      labels:
        app: app-pod
    spec:
      containers:
      - name: app-container
        image: uopsdod/k8s-hostname-amd64-beta:v1
        ports: 
        - containerPort: 80
```

#### Deployment のデプロイ
```bash
# Deployment をデプロイ
$ kubectl apply -f simple-deployment.yaml

# Deployment の状態を確認
$ kubectl get deployments
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
app-deployment   3/3     3            3           19s

# ReplicaSet の状態を確認
$ kubectl get rs
NAME                        DESIRED   CURRENT   READY   AGE
app-deployment-8687d78656   3         3         3       28s

# Pod の状態を確認
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-8687d78656-5bf82   1/1     Running   0          31s
app-deployment-8687d78656-l4ctt   1/1     Running   0          31s
app-deployment-8687d78656-lv7wl   1/1     Running   0          31s

# すべてのリソースを確認
$ kubectl get all
NAME                                  READY   STATUS    RESTARTS   AGE
pod/app-deployment-8687d78656-5bf82   1/1     Running   0          37s
pod/app-deployment-8687d78656-l4ctt   1/1     Running   0          37s
pod/app-deployment-8687d78656-lv7wl   1/1     Running   0          37s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   51m

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/app-deployment   3/3     3            3           37s

NAME                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/app-deployment-8687d78656   3         3         3       37s
```

## Deployment 状態維持メカニズムのテスト

#### Pod 障害のシミュレーション
```bash
# すべての Pod を削除して Deployment の自動再構築機能をテスト
$ kubectl delete pods --all
pod "app-deployment-8687d78656-5bf82" deleted
pod "app-deployment-8687d78656-l4ctt" deleted
pod "app-deployment-8687d78656-lv7wl" deleted

# Deployment が自動的に新しい Pod を作成することを観察
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-8687d78656-2hhtc   1/1     Running   0          37s
app-deployment-8687d78656-4hcvv   1/1     Running   0          37s
app-deployment-8687d78656-4xfcd   1/1     Running   0          37s
```

#### ReplicaSet 削除のテスト
```bash
# すべての ReplicaSet を削除
$ kubectl delete rs --all
replicaset.apps "app-deployment-8687d78656" deleted

# Deployment が自動的に新しい ReplicaSet を作成することを観察
$ kubectl get rs
NAME                        DESIRED   CURRENT   READY   AGE
app-deployment-8687d78656   3         3         3       32s

# Pod がまだ存在することを確認
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-8687d78656-6hzmm   1/1     Running   0          78s
app-deployment-8687d78656-865xb   1/1     Running   0          78s
app-deployment-8687d78656-qm75q   1/1     Running   0          78s
```

## リソースのクリーンアップ
```bash
# すべての Deployment を削除
$ kubectl delete deployments --all
```

## 説明

- **Deployment**: Pod と ReplicaSet の宣言的更新を提供し、Kubernetes で最も一般的に使用されるワークロードリソース
- **replicas: 3**: 維持する Pod レプリカ数を指定
- **selector**: この Deployment に属する Pod を識別する方法を定義
- **template**: コンテナイメージ、ポート、その他の設定を含む Pod の仕様を定義
- **自己修復**: Pod が削除または障害が発生した場合、Deployment は指定されたレプリカ数を維持するために自動的に新しい Pod を作成
- **kubectl get deployments**: Deployment の状態を確認し、準備完了レプリカ数、最新レプリカ数などの情報を表示

## 重要な概念

- **Deployment vs ReplicaSet**: Deployment が ReplicaSet を管理し、ReplicaSet が Pod を管理する階層的管理構造を形成
- **期待状態 vs 実際状態**: Deployment は継続的に監視し、実際に実行されている Pod の数が期待されるレプリカ数と一致することを保証
- **ラベルセレクター**: `matchLabels` を使用してこの Deployment に属する Pod を識別・管理
- **Pod テンプレート**: 新しく作成される Pod が持つべき仕様と設定を定義
- **バージョン管理**: Deployment はローリングアップデートとバージョンロールバック機能をサポート（高度な機能）
- **リソース階層**: Deployment > ReplicaSet > Pod、上位リソースの削除は下位リソースに影響 