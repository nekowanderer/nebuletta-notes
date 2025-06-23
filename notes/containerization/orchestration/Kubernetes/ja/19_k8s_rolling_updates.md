# Kubernetes Rolling Updates の実装

[English](../en/19_k8s_rolling_updates.md) | [繁體中文](../zh-tw/19_k8s_rolling_updates.md) | [日本語](../ja/19_k8s_rolling_updates.md) | [インデックスに戻る](../README.md)

## 前提条件

### Docker サービスの開始
```bash
$ sudo service docker start
$ docker ps
```

### Minikube クラスターの作成
```bash
$ minikube start --driver docker
$ minikube status
```

## 初期バージョン（v1）のデプロイ

### Deployment 定義ファイルの作成
Rolling Update 用の deployment ファイルをコピーして作成：

```bash
$ cp advanced-deployment.yaml advanced-deployment-rollingupdate.yaml
$ cat advanced-deployment-rollingupdate.yaml
```

### v1 バージョンのデプロイ
```bash
$ kubectl apply -f advanced-deployment-rollingupdate.yaml

$ kubectl get deployments
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
app-deployment   3/3     3            3           22s

$ kubectl get rs
NAME                        DESIRED   CURRENT   READY   AGE
app-deployment-7f57cfb4b6   3         3         3       24s
```

### Pod ステータスの確認
```bash
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-7f57cfb4b6-5nmhg   1/1     Running   0          48s
app-deployment-7f57cfb4b6-qns74   1/1     Running   0          48s
app-deployment-7f57cfb4b6-stpv9   1/1     Running   0          48s

$ kubectl describe pod [pod_id] | grep Image:
    Image:          uopsdod/k8s-hostname-amd64-beta:v1
```

## Rolling Update 戦略の設定

### Deployment 設定の変更
`advanced-deployment-rollingupdate.yaml` ファイルを編集して、Rolling Update 戦略を追加し、イメージバージョンを更新します：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
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
        image: uopsdod/k8s-hostname-amd64-beta:v2
        imagePullPolicy: Always
        ports:
        - containerPort: 80
        startupProbe:
          httpGet:
            path: /healthcheck
            port: 80
          initialDelaySeconds: 0
          timeoutSeconds: 1
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 30
        livenessProbe:
          httpGet:
            path: /healthcheck
            port: 80
          initialDelaySeconds: 0
          timeoutSeconds: 1
          periodSeconds: 1
          successThreshold: 1
          failureThreshold: 5
        readinessProbe:
          httpGet:
            path: /healthcheck_dependency
            port: 80
          initialDelaySeconds: 0
          timeoutSeconds: 1
          periodSeconds: 1
          successThreshold: 5
          failureThreshold: 7
```

### 設定の確認
```bash
$ cat advanced-deployment-rollingupdate.yaml
```

## Rolling Update の実行

### v2 バージョンのデプロイ
```bash
$ kubectl apply -f advanced-deployment-rollingupdate.yaml
```

### Rolling Update プロセスのモニタリング
新しいターミナルウィンドウで以下のコマンドを実行して、ReplicaSet の変化をリアルタイムでモニタリングします：

```bash
$ kubectl get rs -w
NAME                        DESIRED   CURRENT   READY   AGE
app-deployment-7f57cfb4b6   3         3         3       3m12s
app-deployment-6f975d88d9   1         0         0       0s
app-deployment-6f975d88d9   1         0         0       0s
app-deployment-6f975d88d9   1         1         0       0s
app-deployment-6f975d88d9   1         1         1       14s
app-deployment-7f57cfb4b6   2         3         3       4m41s
app-deployment-6f975d88d9   2         1         1       14s
app-deployment-7f57cfb4b6   2         3         3       4m41s
app-deployment-6f975d88d9   2         1         1       14s
app-deployment-6f975d88d9   2         2         1       14s
app-deployment-7f57cfb4b6   2         2         2       4m41s
app-deployment-6f975d88d9   2         2         2       29s
app-deployment-7f57cfb4b6   1         2         2       4m56s
app-deployment-7f57cfb4b6   1         2         2       4m56s
app-deployment-6f975d88d9   3         2         2       29s
app-deployment-7f57cfb4b6   1         1         1       4m56s
app-deployment-6f975d88d9   3         2         2       29s
app-deployment-6f975d88d9   3         3         2       29s
app-deployment-6f975d88d9   3         3         3       44s
app-deployment-7f57cfb4b6   0         1         1       5m11s
app-deployment-7f57cfb4b6   0         1         1       5m11s
app-deployment-7f57cfb4b6   0         0         0       5m11s

$ kubectl get pods -w
NAME                              READY   STATUS              RESTARTS   AGE
app-deployment-6f975d88d9-t74gf   0/1     ContainerCreating   0          5s
app-deployment-7f57cfb4b6-5nmhg   1/1     Running             0          4m32s
app-deployment-7f57cfb4b6-qns74   1/1     Running             0          4m32s
app-deployment-7f57cfb4b6-stpv9   1/1     Running             0          4m32s
app-deployment-6f975d88d9-t74gf   0/1     Running             0          10s
app-deployment-6f975d88d9-t74gf   0/1     Running             0          11s
app-deployment-6f975d88d9-t74gf   1/1     Running             0          14s
app-deployment-7f57cfb4b6-stpv9   1/1     Terminating         0          4m41s
app-deployment-6f975d88d9-cq2xk   0/1     Pending             0          0s
app-deployment-6f975d88d9-cq2xk   0/1     Pending             0          0s
app-deployment-6f975d88d9-cq2xk   0/1     ContainerCreating   0          0s
app-deployment-6f975d88d9-cq2xk   0/1     Running             0          3s
app-deployment-6f975d88d9-cq2xk   0/1     Running             0          10s
app-deployment-6f975d88d9-cq2xk   1/1     Running             0          15s
app-deployment-7f57cfb4b6-qns74   1/1     Terminating         0          4m56s
app-deployment-6f975d88d9-2lwcb   0/1     Pending             0          0s
app-deployment-6f975d88d9-2lwcb   0/1     Pending             0          0s
app-deployment-6f975d88d9-2lwcb   0/1     ContainerCreating   0          0s
app-deployment-6f975d88d9-2lwcb   0/1     Running             0          3s
app-deployment-6f975d88d9-2lwcb   0/1     Running             0          10s
app-deployment-7f57cfb4b6-stpv9   0/1     Error               0          5m11s
app-deployment-6f975d88d9-2lwcb   1/1     Running             0          15s
app-deployment-7f57cfb4b6-5nmhg   1/1     Terminating         0          5m11s
app-deployment-7f57cfb4b6-stpv9   0/1     Error               0          5m11s
app-deployment-7f57cfb4b6-stpv9   0/1     Error               0          5m11s
app-deployment-7f57cfb4b6-qns74   0/1     Error               0          5m27s
app-deployment-7f57cfb4b6-5nmhg   0/1     Error               0          5m42s
```

### 更新結果の確認
```bash
# ReplicaSet ステータスの確認
$ kubectl get rs
NAME                        DESIRED   CURRENT   READY   AGE
app-deployment-6f975d88d9   3         3         3       114s
app-deployment-7f57cfb4b6   0         0         0       6m21s

# Pod ステータスの確認
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-6f975d88d9-2lwcb   1/1     Running   0          81s
app-deployment-6f975d88d9-cq2xk   1/1     Running   0          96s
app-deployment-6f975d88d9-t74gf   1/1     Running   0          110s

# Pod が使用するイメージバージョンの確認
$ kubectl describe pod [pod_id] | grep Image:
    Image:          uopsdod/k8s-hostname-amd64-beta:v2
```

## Rolling Update 戦略の説明

### 戦略パラメータ
- **type: RollingUpdate**: Rolling Update 戦略の使用を指定
- **maxSurge: 1**: 希望するレプリカ数を超えて許可される最大数（同時に実行できる Pod 数）
- **maxUnavailable: 0**: 更新中に利用不可能な Pod の数（0 はサービス中断を許可しないことを意味）

### 更新フロー
1. Kubernetes は最初に新しい ReplicaSet を作成して v2 バージョンをデプロイ
2. 段階的に古いバージョンの Pod を新しいバージョンに置き換え
3. 更新プロセス中にサービスが継続的に利用可能であることを保証

## 重要な概念

- **Rolling Update**: サービスを中断することなく段階的に Pod を更新する戦略
- **maxSurge**: 更新プロセス中に希望するレプリカ数を超えることができる数を制御
- **maxUnavailable**: 更新プロセス中に利用不可能になることが許可される Pod の数を制御
- **ReplicaSet 管理**: 各バージョンが対応する Pod を管理するための新しい ReplicaSet を生成
- **ヘルスチェック**: Probe を使用して新しい Pod が正常に動作していることを確認してから次の更新ステップに進む
