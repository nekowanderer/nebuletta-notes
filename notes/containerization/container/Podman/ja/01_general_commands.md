# Podman 基本コマンドガイド

[English](../en/01_general_commands.md) | [繁體中文](../zh-tw/01_general_commands.md) | [日本語](../ja/01_general_commands.md) | [インデックスに戻る](../README.md)

## Podman のインストール

- 公式サイトの[インストールページ](https://podman.io/docs/installation)にアクセスし、使用するプラットフォームに応じてインストール

```bash
$ podman --version
podman version 5.5.2

$ podman machine init
Looking up Podman Machine image at quay.io/podman/machine-os:5.5 to create VM
Getting image source signatures
Copying blob 1f5c0ec86103 done   |
Copying config 44136fa355 done   |
Writing manifest to image destination
1f5c0ec861031a3e51ae30af3a34009ca344437609915ecf7833897e2292b448
Extracting compressed file: podman-machine-default-arm64.raw: done
Machine init complete
To start your machine run:

	podman machine start

$ podman machine start
Starting machine "podman-machine-default"

This machine is currently configured in rootless mode. If your containers
require root permissions (e.g. ports < 1024), or if you run into compatibility
issues with non-podman clients, you can switch using the following command:

	podman machine set --rootful

API forwarding listening on: /var/folders/kl/z4yft4256c70_b6154gqzgqc0000gn/T/podman/podman-machine-default-api.sock

Another process was listening on the default Docker API socket address.
You can still connect Docker API clients by setting DOCKER_HOST using the
following command in your terminal session:

        export DOCKER_HOST='unix:///var/folders/kl/z4yft4256c70_b6154gqzgqc0000gn/T/podman/podman-machine-default-api.sock'

Machine "podman-machine-default" started successfully
```

## 基本操作

### ローカルイメージの確認
```bash
$ podman images
```

### イメージのプル
```bash
$ podman pull alpine:3.16
$ podman images
```

### コンテナの実行

#### 一回限りのコマンド実行
```bash
$ podman run alpine:3.16 echo "test"
$ podman run alpine:3.16 df
$ podman run alpine:3.16 ls /
```

#### インタラクティブ実行
```bash
$ podman run -it alpine:3.16 /bin/sh
```

インタラクティブシェル内で実行可能：
```bash
echo "test"
df
ls
exit
```

### コンテナ管理

#### 実行中のコンテナを確認
```bash
$ podman container ls
```

#### バックグラウンドでコンテナを実行
```bash
$ podman run -d -it --name c001 alpine:3.16 /bin/sh
$ podman container ls
```

#### 実行中のコンテナに入る
```bash
$ podman exec -it c001 /bin/sh
```

コンテナ内で実行可能：
```bash
df
exit
```

## クリーンアップ操作

### コンテナの停止と削除

#### 個別コンテナ操作
```bash
$ podman container stop c001
$ podman container ls
$ podman container ls -a
$ podman container rm c001
$ podman container ls -a
```

#### 全コンテナの一括クリーンアップ
```bash
# -q は ID のみを表示
$ podman container stop $(podman container ls -q)
$ podman container rm $(podman container ls -a -q)
```

### イメージの削除

#### 個別イメージ削除
```bash
$ podman images
$ podman rmi alpine:3.16
$ podman images
```

#### 全イメージの一括削除
```bash
$ podman rmi $(podman images -q)
$ podman images
```

## カスタムイメージの作成

### Dockerfile の作成
```bash
$ vi Dockerfile
```

Dockerfile の内容：
```dockerfile
FROM alpine:3.16
RUN apk --update add apache2
RUN rm -rf /var/cache/apk/*
ENTRYPOINT ["httpd", "-D", "FOREGROUND"]
```

### イメージのビルド
```bash
$ cat Dockerfile
$ podman build -t podman-apache-image .
$ podman images
```

### カスタムイメージの実行
```bash
$ podman run -d -p 8081:80 --name c002 podman-apache-image
$ podman container ls

# テスト
$ curl -X GET localhost:8081
<html><body><h1>It works!</h1></body></html>
```

## Docker Hub へのプッシュ

### プッシュの準備
```bash
$ podman build -t your_docker_hub_id/podman-apache-image .
$ podman images
```

### Docker Hub へのログイン
```bash
$ podman logout docker.io
$ podman login docker.io
```

### イメージのプッシュ
```bash
$ podman push your_docker_hub_id/podman-apache-image
```

> プッシュしたイメージは [Docker Hub](https://hub.docker.com/) で確認できます

## カスタム Docker Hub イメージの使用

### Docker Hub からのプルと実行
```bash
$ podman run -d -p 8082:80 --name c003 docker.io/your_docker_hub_id/podman-apache-image
$ podman container ls

# テスト
$ curl -X GET localhost:8081
<html><body><h1>It works!</h1></body></html>
```