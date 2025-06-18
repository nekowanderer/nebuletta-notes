# MiniKube インストールと利用ガイド

[English](../en/05_minikube_basics.md) | [繁體中文](../zh-tw/05_minikube_basics.md) | [日本語](../ja/05_minikube_basics.md) | [インデックスに戻る](../README.md)

## 目次
1. [環境準備](#環境準備)
2. [Docker のインストール](#docker-のインストール)
3. [MiniKube のインストール](#minikube-のインストール)
4. [クラスタの作成と管理](#クラスタの作成と管理)
5. [アプリケーションのデプロイ](#アプリケーションのデプロイ)
6. [サービス管理](#サービス管理)
7. [ネットワーク接続テスト](#ネットワーク接続テスト)
8. [リソースのクリーンアップ](#リソースのクリーンアップ)

---

## Docker のインストール

#### 1. Docker パッケージのインストール
```bash
$ sudo yum install docker -y
```

#### 2. ユーザー権限の設定
```bash
$ sudo usermod -aG docker $USER && newgrp docker
```

#### 3. Docker サービスの起動
```bash
$ sudo service docker start
```

#### 4. Docker インストールの確認
```bash
$ docker ps
```

---

## MiniKube のインストール

#### 1. ホームディレクトリに MiniKube バイナリをダウンロード
```bash
$ wget https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
```

#### 2. 実行権限の付与とインストール
```bash
$ chmod +x minikube-linux-amd64
$ sudo mv minikube-linux-amd64 /usr/local/bin/minikube
```

#### 3. インストールの確認
```bash
$ minikube version
minikube version: v1.36.0

$ minikube status
Profile "minikube" not found. Run "minikube profile list" to view all profiles.
To start a cluster, run: "minikube start"
```

---

## クラスタの作成と管理

#### 1. MiniKube クラスタの起動
`driver` オプションで計算リソースの起動先を決定します。minikube では、マスターノードとワーカーノードは基本的に同じ場所にあります。
```bash
$ minikube start --driver docker
```

#### 2. クラスタの状態確認
```bash
$ docker ps
# ...docker コンテナ情報...
```
このコマンドで minikube という名前の docker コンテナが起動していることが分かります。K8S はこのコンテナ内で動作します（マスター・ワーカー兼用）。

クラスタを維持するため、いくつかの Pod がすでに稼働しています。以下で確認できます：
```bash
$ minikube kubectl -- get pods -A
# ...Pod 一覧...
```

#### 3. kubectl エイリアスの設定
```bash
vi ~/.bashrc
alias kubectl='minikube kubectl -- '
source ~/.bashrc
```

#### 4. kubectl エイリアスの確認
```bash
kubectl get pods -A
```

#### 5. ノード情報の確認
```bash
$ kubectl get nodes
# ...ノード情報...
```

特定ノードの詳細情報：
```bash
$ kubectl describe nodes minikube
# ...詳細ノード情報...
```

---

## アプリケーションのデプロイ

#### 1. Deployment の作成
```bash
$ kubectl create deployment hello-minikube --image=k8s.gcr.io/echoserver:1.10
```

#### 2. Deployment 状態の確認
```bash
$ kubectl get deployments
# ...Deployment 情報...
```

詳細確認：
```bash
$ kubectl describe deployments hello-minikube
# ...詳細 Deployment 情報...
```

#### 3. Pod の起動確認
```bash
$ kubectl get pods
# ...Pod 情報...
```

```bash
$ kubectl describe pods [pod_name]
# ...詳細 Pod 情報...
```

#### 4. Pod の配置確認
```bash
$ kubectl get nodes
# ...ノード情報...
```

```bash
$ kubectl describe nodes minikube
# ...Pod 割り当て情報...
```

---

## サービス管理

#### 1. サービスの公開
```bash
$ kubectl expose deployment hello-minikube --port=8080
service/hello-minikube exposed
```

#### 2. サービス状態の確認
```bash
$ kubectl get services
# ...サービス情報...
```

詳細確認：
```bash
$ kubectl describe services hello-minikube
# ...詳細サービス情報...
```

---

## ネットワーク接続テスト

#### 1. コンテナ内での接続テスト
```bash
$ docker ps
# ...minikube コンテナ情報...
```

```bash
$ docker exec -it minikube bash
root@minikube:/$ curl 10.100.51.36:8080
# ...レスポンス...
```

#### 2. ポートフォワーディングの設定
```bash
$ kubectl port-forward service/hello-minikube 8081:8080 &
# ...フォワーディング情報...
```

#### 3. ローカル接続テスト
```bash
$ curl localhost:8081
# ...レスポンス...
```

#### 4. ポートフォワーディングの停止
```bash
$ ps
$ kill [kubectl_PID]
$ ps
```

---

## リソースのクリーンアップ

#### 1. サービスの削除
```bash
$ kubectl get services
$ kubectl delete service hello-minikube
$ kubectl get service
```

#### 2. Deployment の削除
```bash
$ kubectl get deployments
$ kubectl delete deployment hello-minikube
$ kubectl get deployments
$ kubectl get pods
```

#### 3. クラスタの停止と削除（任意）
```bash
$ minikube status
$ minikube stop
$ minikube delete
$ minikube status
```

---

#### クラスタ管理
- `minikube start` - クラスタの起動
- `minikube stop` - クラスタの停止
- `minikube delete` - クラスタの削除
- `minikube status` - クラスタ状態の確認

#### Pod 管理
- `kubectl get pods` - Pod 一覧
- `kubectl describe pod [pod_name]` - Pod 詳細
- `kubectl logs [pod_name]` - Pod ログ

#### サービス管理
- `kubectl get services` - サービス一覧
- `kubectl expose deployment [name] --port=[port]` - サービス作成
- `kubectl delete service [name]` - サービス削除

#### ネットワーク
- `kubectl port-forward service/[service_name] [local_port]:[service_port]` - ポートフォワーディング

---

## 注意事項

1. **権限設定**: docker グループにユーザーが追加されていることを確認してください
2. **ネットワーク**: Docker イメージのダウンロードには安定したインターネット接続が必要です
3. **リソース**: MiniKube には十分なメモリと CPU が必要です
4. **ファイアウォール**: 必要なポートが開放されていることを確認してください 