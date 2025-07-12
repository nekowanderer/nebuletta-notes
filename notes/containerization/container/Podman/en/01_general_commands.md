# Podman Basic Commands Guide

[English](../en/01_general_commands.md) | [繁體中文](../zh-tw/01_general_commands.md) | [日本語](../ja/01_general_commands.md) | [Back to Index](../README.md)

## Installing Podman

- Visit the official website's [installation page](https://podman.io/docs/installation) and install according to your platform

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

## Basic Operations

### View Local Images
```bash
$ podman images
```

### Pull Images
```bash
$ podman pull alpine:3.16
$ podman images
```

### Run Containers

#### One-time Command Execution
```bash
$ podman run alpine:3.16 echo "test"
$ podman run alpine:3.16 df
$ podman run alpine:3.16 ls /
```

#### Interactive Execution
```bash
$ podman run -it alpine:3.16 /bin/sh
```

In the interactive shell, you can execute:
```bash
echo "test"
df
ls
exit
```

### Container Management

#### View Running Containers
```bash
$ podman container ls
```

#### Run Container in Background
```bash
$ podman run -d -it --name c001 alpine:3.16 /bin/sh
$ podman container ls
```

#### Enter Running Container
```bash
$ podman exec -it c001 /bin/sh
```

Inside the container, you can execute:
```bash
df
exit
```

## Cleanup Operations

### Stop and Remove Containers

#### Individual Container Operations
```bash
$ podman container stop c001
$ podman container ls
$ podman container ls -a
$ podman container rm c001
$ podman container ls -a
```

#### Batch Cleanup of All Containers
```bash
# -q means list IDs only
$ podman container stop $(podman container ls -q)
$ podman container rm $(podman container ls -a -q)
```

### Remove Images

#### Individual Image Removal
```bash
$ podman images
$ podman rmi alpine:3.16
$ podman images
```

#### Batch Removal of All Images
```bash
$ podman rmi $(podman images -q)
$ podman images
```

## Building Custom Images

### Create Dockerfile
```bash
$ vi Dockerfile
```

Dockerfile content:
```dockerfile
FROM alpine:3.16
RUN apk --update add apache2
RUN rm -rf /var/cache/apk/*
ENTRYPOINT ["httpd", "-D", "FOREGROUND"]
```

### Build Image
```bash
$ cat Dockerfile
$ podman build -t podman-apache-image .
$ podman images
```

### Run Custom Image
```bash
$ podman run -d -p 8081:80 --name c002 podman-apache-image
$ podman container ls

# Test
$ curl -X GET localhost:8081
<html><body><h1>It works!</h1></body></html>
```

## Push to Docker Hub

### Prepare for Push
```bash
$ podman build -t your_docker_hub_id/podman-apache-image .
$ podman images
```

### Login to Docker Hub
```bash
$ podman logout docker.io
$ podman login docker.io
```

### Push Image
```bash
$ podman push your_docker_hub_id/podman-apache-image
```

> You can view the pushed image on [Docker Hub](https://hub.docker.com/)

## Using Custom Docker Hub Images

### Pull and Run from Docker Hub
```bash
$ podman run -d -p 8082:80 --name c003 docker.io/your_docker_hub_id/podman-apache-image
$ podman container ls

# Test
$ curl -X GET localhost:8081
<html><body><h1>It works!</h1></body></html>
```