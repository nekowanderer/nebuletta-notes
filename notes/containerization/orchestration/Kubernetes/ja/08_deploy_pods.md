# Kubernetes Pod デプロイメント実践

[English](../en/08_deploy_pods.md) | [繁體中文](../zh-tw/08_deploy_pods.md) | [日本語](../ja/08_deploy_pods.md) | [インデックスに戻る](../README.md)

## 前提条件

#### Docker サービスの起動
```bash
sudo service docker start
docker ps
```

#### Minikube クラスターの作成
```bash
minikube start --driver docker
minikube status
```

## Pod の作成

#### Pod 定義ファイルの作成
`simple-pod.yaml` ファイルを作成：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app-container
    image: uopsdod/k8s-hostname-amd64-beta:v1
    ports:
    - containerPort: 80
```

#### Pod のデプロイ
```bash
# Pod をデプロイ
kubectl apply -f simple-pod.yaml

# Pod の状態を確認
kubectl get pods

# Pod の状態変化をリアルタイムで監視
kubectl get pods -w
```

## リソースのクリーンアップ
```bash
# すべての Pod を削除
kubectl delete pods --all
```

## 説明

- **Pod**: Kubernetes における最小のデプロイメント単位で、1つまたは複数のコンテナを含むことができます
- **kubectl apply**: Kubernetes リソースの作成または更新
- **kubectl get pods -w**: または `--watch`、Pod の状態をリアルタイムで監視、Ctrl+C で監視を停止
- **kubectl delete pods --all**: クリーンアップのためにすべての Pod を削除 