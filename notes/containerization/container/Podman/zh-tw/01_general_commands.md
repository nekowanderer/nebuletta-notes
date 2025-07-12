# Podman 基本指令指南

[English](../en/01_general_commands.md) | [繁體中文](../zh-tw/01_general_commands.md) | [日本語](../ja/01_general_commands.md) | [回到索引](../README.md)

## 安裝 Podman

- 至官網的[安裝頁面](https://podman.io/docs/installation)，根據要使用的平台安裝

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

### 查看本地映像檔
```bash
$ podman images
```

### 拉取映像檔
```bash
$ podman pull alpine:3.16
$ podman images
```

### 執行容器

#### 一次性執行指令
```bash
$ podman run alpine:3.16 echo "test"
$ podman run alpine:3.16 df
$ podman run alpine:3.16 ls /
```

#### 互動式執行
```bash
$ podman run -it alpine:3.16 /bin/sh
```

在互動式 shell 中可以執行：
```bash
echo "test"
df
ls
exit
```

### 容器管理

#### 查看執行中的容器
```bash
$ podman container ls
```

#### 在背景執行容器
```bash
$ podman run -d -it --name c001 alpine:3.16 /bin/sh
$ podman container ls
```

#### 進入執行中的容器
```bash
$ podman exec -it c001 /bin/sh
```

在容器內可以執行：
```bash
df
exit
```

## 清理操作

### 停止和移除容器

#### 個別容器操作
```bash
$ podman container stop c001
$ podman container ls
$ podman container ls -a
$ podman container rm c001
$ podman container ls -a
```

#### 批次清理所有容器
```bash
# -q 表示列出 id
$ podman container stop $(podman container ls -q)
$ podman container rm $(podman container ls -a -q)
```

### 移除映像檔

#### 個別映像檔移除
```bash
$ podman images
$ podman rmi alpine:3.16
$ podman images
```

#### 批次移除所有映像檔
```bash
$ podman rmi $(podman images -q)
$ podman images
```

## 自建映像檔

### 建立 Dockerfile
```bash
$ vi Dockerfile
```

Dockerfile 內容：
```dockerfile
FROM alpine:3.16
RUN apk --update add apache2
RUN rm -rf /var/cache/apk/*
ENTRYPOINT ["httpd", "-D", "FOREGROUND"]
```

### 建置映像檔
```bash
$ cat Dockerfile
$ podman build -t podman-apache-image .
$ podman images
```

### 執行自建映像檔
```bash
$ podman run -d -p 8081:80 --name c002 podman-apache-image
$ podman container ls

# 測試
$ curl -X GET localhost:8081
<html><body><h1>It works!</h1></body></html>
```

## 推送到 Docker Hub

### 準備推送
```bash
$ podman build -t your_docker_hub_id/podman-apache-image .
$ podman images
```

### 登入 Docker Hub
```bash
$ podman logout docker.io
$ podman login docker.io
```

### 推送映像檔
```bash
$ podman push your_docker_hub_id/podman-apache-image
```

> 可以在 [Docker Hub](https://hub.docker.com/) 查看推送的映像檔

## 使用自建的 Docker Hub 映像檔

### 從 Docker Hub 拉取並執行
```bash
$ podman run -d -p 8082:80 --name c003 docker.io/your_docker_hub_id/podman-apache-image
$ podman container ls

# 測試
$ curl -X GET localhost:8081
<html><body><h1>It works!</h1></body></html>
```
