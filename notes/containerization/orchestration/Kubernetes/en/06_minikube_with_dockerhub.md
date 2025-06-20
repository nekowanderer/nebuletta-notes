# Minikube Integration with Dockerhub

[English](../en/06_minikube_with_dockerhub.md) | [繁體中文](../zh-tw/06_minikube_with_dockerhub.md) | [日本語](../ja/06_minikube_with_dockerhub.md) | [Back to Index](../README.md)

This guide demonstrates how to create a Docker image, upload it to Dockerhub, and deploy applications in Minikube.

## Prerequisites

### 1. Install and Start Docker

```bash
# Install Docker
sudo yum install docker -y

# Add user to docker group and reload groups
sudo usermod -aG docker $USER && newgrp docker

# Start Docker service
sudo service docker start

# Verify Docker is running
docker ps
```

## Create Docker Image

### 2. Create Dockerfile

```bash
# Create Dockerfile
vi Dockerfile
```

Dockerfile content:
```dockerfile
FROM alpine:3.14
WORKDIR /var/www/localhost/htdocs
RUN apk --update add apache2
RUN rm -rf /var/cache/apk/*
RUN echo "<h1>Application on K8S Demo <h1>" >> index.html
ENTRYPOINT ["httpd","-D","FOREGROUND"]
```

```bash
# View Dockerfile content
cat Dockerfile
```

### 3. Build Docker Image

```bash
# Build image
docker build -t your_docker_hub_account/k8sgithub001 .

# View local images
docker images
```

### 4. Test Docker Container

```bash
# Start container for testing
docker run -d -p 8081:80 --name k8sgithub001 your_docker_hub_account/k8sgithub001 

# View running containers
docker ps

# Test application
curl localhost:8081

# Clean up test container
docker stop k8sgithub001
docker rm k8sgithub001
```

## Upload to Dockerhub

### 5. Login and Upload Image

```bash
# Login to Dockerhub
docker login

# View local images
docker images

# Upload image to Dockerhub
docker push your_docker_hub_account/k8sgithub001 
```

### 6. Verify Upload Success

Visit [Dockerhub](https://hub.docker.com/) to confirm the image has been successfully uploaded.

## Deploy in Minikube

### 7. Start Minikube Cluster

```bash
# Start Minikube (using Docker driver)
minikube start --driver docker

# Check cluster status
minikube status
```

### 8. Deploy First Application

```bash
# Create deployment
kubectl create deployment k8sgithub001 --image=your_docker_hub_account/k8sgithub001

# View deployments
kubectl get deployments

# View pods
kubectl get pods
```

### 9. Create Service and Enable External Access

```bash
# Create service
kubectl expose deployment k8sgithub001 --port=80

# View services
kubectl get services

# Enable external access (run in background)
kubectl port-forward service/k8sgithub001 8081:80 &

# Test application
curl localhost:8081
```

### 10. Deploy Second Application

```bash
# Create second deployment (using someone else's image)
kubectl create deployment k8s-hostname-001 --image=uopsdod/k8s-hostname-amd64-beta

# View deployments
kubectl get deployments

# View pods
kubectl get pods

# Expose port
kubectl expose deployment k8s-hostname-001 --port=80

# View services
kubectl get services

# Port-forwarding
kubectl port-forward service/k8s-hostname-001 8082:80 &

# Test application
curl localhost:8082
```

### 11. Manage Port-forward Processes

```bash
# View background processes
ps

# Stop port-forward (replace [kubectl PID] with actual PID)
kill [kubectl PID]
kill [kubectl PID]

# Confirm processes have stopped
ps
```

## Clean Up Resources

### 12. Delete Services

```bash
# View all services
kubectl get services

# Delete all services
kubectl delete services --all
```

### 13. Delete Deployments

```bash
# View all deployments
kubectl get deployments

# Delete all deployments
kubectl delete deployment --all
```

### 14. Stop and Delete Minikube Cluster

```bash
# Stop cluster
minikube stop

# Delete cluster
minikube delete

# Verify cluster status
minikube status
```

## Summary

Complete workflow:
1. Create Docker image
2. Upload to Dockerhub
3. Deploy applications in Minikube
4. Create services and enable external access
5. Clean up resources

This workflow serves as a basic example for deploying applications in a Kubernetes environment. 