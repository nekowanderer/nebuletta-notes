# Kubernetes ReplicaSet デプロイ実践

[English](../en/09_deploy_replica_set.md) | [繁體中文](../zh-tw/09_deploy_replica_set.md) | [日本語](../ja/09_deploy_replica_set.md) | [インデックスに戻る](../README.md)

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

## ReplicaSet の作成

#### ReplicaSet 定義ファイルの作成
`simple-replicaset.yaml` ファイルを作成：

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: app-rs
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

#### ReplicaSet のデプロイ
```bash
# ReplicaSet をデプロイ
$ kubectl apply -f simple-replicaset.yaml

# ReplicaSet の状態を確認
$ kubectl get rs
NAME     DESIRED   CURRENT   READY   AGE
app-rs   3         3         3       26s

# Pod の状態を確認、Pod の起動を待機
$ kubectl get pods
NAME           READY   STATUS    RESTARTS   AGE
app-rs-24r5n   1/1     Running   0          32s
app-rs-7fswq   1/1     Running   0          32s
app-rs-bhc7x   1/1     Running   0          32s
```

## ReplicaSet の自己修復メカニズムのテスト

#### Pod 障害のシミュレーション
```bash
# すべての Pod を削除して ReplicaSet の自動再構築をテスト
$ kubectl delete pods --all
pod "app-rs-24r5n" deleted
pod "app-rs-7fswq" deleted
pod "app-rs-bhc7x" deleted

# ReplicaSet が自動的に新しい Pod を作成することを観察
$ kubectl get pods
NAME           READY   STATUS    RESTARTS   AGE
app-rs-4mb4x   1/1     Running   0          35s
app-rs-rjdvn   1/1     Running   0          35s
app-rs-vkct7   1/1     Running   0          35s
```

## リソースのクリーンアップ
```bash
# すべての ReplicaSet を削除
$ kubectl delete rs --all
```

## 説明

- **ReplicaSet**: 指定された数の Pod レプリカが継続的に実行されることを保証し、高可用性と負荷分散を提供
- **replicas: 3**: 維持する Pod レプリカ数を指定
- **selector**: この ReplicaSet に属する Pod を識別する方法を定義
- **template**: Pod の仕様を定義し、コンテナイメージ、ポート、その他の設定を含む
- **自己修復**: Pod が削除または障害が発生した場合、ReplicaSet は指定されたレプリカ数を維持するために自動的に新しい Pod を作成
- **kubectl get rs**: ReplicaSet の状態を表示し、希望するレプリカ数、現在のレプリカ数、その他の情報を表示

## 重要な概念

- **希望状態 vs 実際の状態**: ReplicaSet は継続的に監視し、実際に実行されている Pod の数が希望するレプリカ数と一致することを保証
- **ラベルセレクター**: `matchLabels` を使用してこの ReplicaSet に属する Pod を識別・管理
- **Pod テンプレート**: 新しく作成される Pod が持つべき仕様と設定を定義 