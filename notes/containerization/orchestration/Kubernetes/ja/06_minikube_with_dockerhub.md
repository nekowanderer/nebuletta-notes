# Minikube と Dockerhub の統合

[English](../en/06_minikube_with_dockerhub.md) | [繁體中文](../zh-tw/06_minikube_with_dockerhub.md) | [日本語](../ja/06_minikube_with_dockerhub.md) | [インデックスに戻る](../README.md)

Docker イメージの作成、Dockerhub へのアップロード、Minikube でのアプリケーション展開方法について説明します。

## 前提条件

### 1. Docker のインストールと起動

```bash
# Docker のインストール
sudo yum install docker -y

# ユーザーを docker グループに追加し、グループを再読み込み
sudo usermod -aG docker $USER && newgrp docker

# Docker サービスの起動
sudo service docker start

# Docker が動作していることを確認
docker ps
```

## Docker イメージの作成

### 2. Dockerfile の作成

```bash
# Dockerfile の作成
vi Dockerfile
```

Dockerfile の内容：
```dockerfile
FROM alpine:3.14
WORKDIR /var/www/localhost/htdocs
RUN apk --update add apache2
RUN rm -rf /var/cache/apk/*
RUN echo "<h1>Application on K8S Demo <h1>" >> index.html
ENTRYPOINT ["httpd","-D","FOREGROUND"]
```

```bash
# Dockerfile の内容を確認
cat Dockerfile
```

### 3. Docker イメージのビルド

```bash
# イメージのビルド
docker build -t your_docker_hub_account/k8sgithub001 .

# ローカルイメージの確認
docker images
```

### 4. Docker コンテナのテスト

```bash
# テスト用コンテナの起動
docker run -d -p 8081:80 --name k8sgithub001 your_docker_hub_account/k8sgithub001 

# 実行中のコンテナの確認
docker ps

# アプリケーションのテスト
curl localhost:8081

# テスト用コンテナのクリーンアップ
docker stop k8sgithub001
docker rm k8sgithub001
```

## Dockerhub へのアップロード

### 5. ログインとイメージのアップロード

```bash
# Dockerhub へのログイン
docker login

# ローカルイメージの確認
docker images

# Dockerhub へのイメージアップロード
docker push your_docker_hub_account/k8sgithub001 
```

### 6. アップロード成功の確認

[Dockerhub](https://hub.docker.com/) にアクセスして、イメージが正常にアップロードされたことを確認してください。

## Minikube での展開

### 7. Minikube クラスターの起動

```bash
# Minikube の起動（Docker ドライバーを使用）
minikube start --driver docker

# クラスターの状態確認
minikube status
```

### 8. 最初のアプリケーションの展開

```bash
# デプロイメントの作成
kubectl create deployment k8sgithub001 --image=your_docker_hub_account/k8sgithub001

# デプロイメントの確認
kubectl get deployments

# ポッドの確認
kubectl get pods
```

### 9. サービスの作成と外部アクセスの有効化

```bash
# サービスの作成
kubectl expose deployment k8sgithub001 --port=80

# サービスの確認
kubectl get services

# 外部アクセスの有効化（バックグラウンドで実行）
kubectl port-forward service/k8sgithub001 8081:80 &

# アプリケーションのテスト
curl localhost:8081
```

### 10. 2番目のアプリケーションの展開

```bash
# 2番目のデプロイメントの作成（他の人のイメージを使用）
kubectl create deployment k8s-hostname-001 --image=uopsdod/k8s-hostname-amd64-beta

# デプロイメントの確認
kubectl get deployments

# ポッドの確認
kubectl get pods

# ポートの公開
kubectl expose deployment k8s-hostname-001 --port=80

# サービスの確認
kubectl get services

# ポートフォワーディング
kubectl port-forward service/k8s-hostname-001 8082:80 &

# アプリケーションのテスト
curl localhost:8082
```

### 11. ポートフォワードプロセスの管理

```bash
# バックグラウンドプロセスの確認
ps

# ポートフォワードの停止（[kubectl PID] を実際の PID に置き換えてください）
kill [kubectl PID]
kill [kubectl PID]

# プロセスが停止したことを確認
ps
```

## リソースのクリーンアップ

### 12. サービスの削除

```bash
# すべてのサービスの確認
kubectl get services

# すべてのサービスの削除
kubectl delete services --all
```

### 13. デプロイメントの削除

```bash
# すべてのデプロイメントの確認
kubectl get deployments

# すべてのデプロイメントの削除
kubectl delete deployment --all
```

### 14. Minikube クラスターの停止と削除

```bash
# クラスターの停止
minikube stop

# クラスターの削除
minikube delete

# クラスターの状態確認
minikube status
```

## まとめ

完全なワークフロー：
1. Docker イメージの作成
2. Dockerhub へのアップロード
3. Minikube でのアプリケーション展開
4. サービスの作成と外部アクセスの有効化
5. リソースのクリーンアップ

このワークフローは、Kubernetes 環境でのアプリケーション展開の基本的な例として活用できます。 
