# Minikube 與 Dockerhub 整合

[English](../en/06_minikube_with_dockerhub.md) | [繁體中文](../zh-tw/06_minikube_with_dockerhub.md) | [日本語](../ja/06_minikube_with_dockerhub.md) | [回到索引](../README.md)

記錄如何建立 Docker Image、上傳至 Dockerhub，並在 Minikube 中部署應用程式。

## 前置準備

### 1. 安裝並啟動 Docker

```bash
# 安裝 Docker
sudo yum install docker -y

# 將使用者加入 docker 群組並重新載入群組
sudo usermod -aG docker $USER && newgrp docker

# 啟動 Docker 服務
sudo service docker start

# 確認 Docker 正在運行
docker ps
```

## 建立 Docker Image

### 2. 建立 Dockerfile

```bash
# 建立 Dockerfile
vi Dockerfile
```

Dockerfile 內容：
```dockerfile
FROM alpine:3.14
WORKDIR /var/www/localhost/htdocs
RUN apk --update add apache2
RUN rm -rf /var/cache/apk/*
RUN echo "<h1>Application on K8S Demo <h1>" >> index.html
ENTRYPOINT ["httpd","-D","FOREGROUND"]
```

```bash
# 查看 Dockerfile 內容
cat Dockerfile
```

### 3. 建立 Docker Image

```bash
# 建立 Image
docker build -t your_docker_hub_account/k8sgithub001 .

# 查看本地 Images
docker images
```

### 4. 測試 Docker Container

```bash
# 啟動 Container 進行測試
docker run -d -p 8081:80 --name k8sgithub001 your_docker_hub_account/k8sgithub001 

# 查看運行中的 Containers
docker ps

# 測試應用程式
curl localhost:8081

# 清理測試用的 Container
docker stop k8sgithub001
docker rm k8sgithub001
```

## 上傳至 Dockerhub

### 5. 登入並上傳 Image

```bash
# 登入 Dockerhub
docker login

# 查看本地 Images
docker images

# 上傳 Image 至 Dockerhub
docker push your_docker_hub_account/k8sgithub001 
```

### 6. 確認上傳成功

前往 [Dockerhub](https://hub.docker.com/) 確認 Image 已成功上傳。

## 在 Minikube 中部署

### 7. 啟動 Minikube Cluster

```bash
# 啟動 Minikube（使用 Docker driver）
minikube start --driver docker

# 檢查 Cluster 狀態
minikube status
```

### 8. 部署第一個應用程式

```bash
# 建立 Deployment
kubectl create deployment k8sgithub001 --image=your_docker_hub_account/k8sgithub001

# 查看 Deployments
kubectl get deployments

# 查看 Pods
kubectl get pods
```

### 9. 建立 Service 並開放連線

```bash
# 建立 Service
kubectl expose deployment k8sgithub001 --port=80

# 查看 Services
kubectl get services

# 開放外部連線（在背景執行）
kubectl port-forward service/k8sgithub001 8081:80 &

# 測試應用程式
curl localhost:8081
```

### 10. 部署第二個應用程式

```bash
# 建立第二個 Deployment (抓別人的 image)
kubectl create deployment k8s-hostname-001 --image=uopsdod/k8s-hostname-amd64-beta

# 查看 Deployments
kubectl get deployments

# 查看 Pods
kubectl get pods

# expose port
kubectl expose deployment k8s-hostname-001 --port=80

# 查看 Services
kubectl get services

# Port-forwarding
kubectl port-forward service/k8s-hostname-001 8082:80 &

# 測試應用程式
curl localhost:8082
```

### 11. 管理 Port-forward 程序

```bash
# 查看背景程序
ps

# 停止 Port-forward（替換 [kubectl PID] 為實際的 PID）
kill [kubectl PID]
kill [kubectl PID]

# 確認程序已停止
ps
```

## 清理資源

### 12. 刪除 Services

```bash
# 查看所有 Services
kubectl get services

# 刪除所有 Services
kubectl delete services --all
```

### 13. 刪除 Deployments

```bash
# 查看所有 Deployments
kubectl get deployments

# 刪除所有 Deployments
kubectl delete deployment --all
```

### 14. 停止並刪除 Minikube Cluster

```bash
# 停止 Cluster
minikube stop

# 刪除 Cluster
minikube delete

# 確認 Cluster 狀態
minikube status
```

## 總結

完整的流程：
1. 建立 Docker Image
2. 上傳至 Dockerhub
3. 在 Minikube 中部署應用程式
4. 建立 Service 並開放外部連線
5. 清理資源

這個流程可以作為在 Kubernetes 環境中部署應用程式的基本範例。
