# MiniKube Installation and Usage Guide

[English](../en/05_minikube_basics.md) | [繁體中文](../zh-tw/05_minikube_basics.md) | [日本語](../ja/05_minikube_basics.md) | [Back to Index](../README.md)

## Table of Contents
1. [Environment Preparation](#environment-preparation)
2. [Install Docker](#install-docker)
3. [Install MiniKube](#install-minikube)
4. [Create and Manage Cluster](#create-and-manage-cluster)
5. [Deploy Applications](#deploy-applications)
6. [Service Management](#service-management)
7. [Network Connectivity Test](#network-connectivity-test)
8. [Resource Cleanup](#resource-cleanup)

---

## Install Docker

#### 1. Install Docker package
```bash
$ sudo yum install docker -y
```

#### 2. Set user permissions
```bash
$ sudo usermod -aG docker $USER && newgrp docker
```

#### 3. Start Docker service
```bash
$ sudo service docker start
```

#### 4. Verify Docker installation
```bash
$ docker ps
```

---

## Install MiniKube

#### 1. Download MiniKube binary to your home directory
```bash
$ wget https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
```

#### 2. Set execution permission and install
```bash
$ chmod +x minikube-linux-amd64
$ sudo mv minikube-linux-amd64 /usr/local/bin/minikube
```

#### 3. Verify installation
```bash
$ minikube version
minikube version: v1.36.0

$ minikube status
Profile "minikube" not found. Run "minikube profile list" to view all profiles.
To start a cluster, run: "minikube start"
```

---

## Create and Manage Cluster

#### 1. Start MiniKube Cluster
Use the driver option to decide where to launch the compute resources. In minikube, master and worker nodes are usually on the same machine.
```bash
$ minikube start --driver docker
```

#### 2. Check Cluster Status
```bash
$ docker ps
# ...docker container info...
```
This means a docker container named minikube is running, and K8S operates inside this container (as both master and worker node).

To keep the cluster running, some pods are already active. You can check them with:
```bash
$ minikube kubectl -- get pods -A
# ...pod list...
```

#### 3. Set kubectl alias
```bash
vi ~/.bashrc
alias kubectl='minikube kubectl -- '
source ~/.bashrc
```

#### 4. Verify kubectl alias
```bash
kubectl get pods -A
```

#### 5. View node information
```bash
$ kubectl get nodes
# ...node info...
```

To see more details for a specific node:
```bash
$ kubectl describe nodes minikube
# ...detailed node info...
```

---

## Deploy Applications

#### 1. Create Deployment
```bash
$ kubectl create deployment hello-minikube --image=k8s.gcr.io/echoserver:1.10
```

#### 2. Check Deployment Status
```bash
$ kubectl get deployments
# ...deployment info...
```

For more details:
```bash
$ kubectl describe deployments hello-minikube
# ...detailed deployment info...
```

#### 3. Wait for Pod to be ready
```bash
$ kubectl get pods
# ...pod info...
```

```bash
$ kubectl describe pods [pod_name]
# ...detailed pod info...
```

#### 4. Check Pod distribution
```bash
$ kubectl get nodes
# ...node info...
```

```bash
$ kubectl describe nodes minikube
# ...pod allocation info...
```

---

## Service Management

#### 1. Expose Service
```bash
$ kubectl expose deployment hello-minikube --port=8080
service/hello-minikube exposed
```

#### 2. Check Service Status
```bash
$ kubectl get services
# ...service info...
```

For more details:
```bash
$ kubectl describe services hello-minikube
# ...detailed service info...
```

---

## Network Connectivity Test

#### 1. Test connection inside container
```bash
$ docker ps
# ...minikube container info...
```

```bash
$ docker exec -it minikube bash
root@minikube:/$ curl 10.100.51.36:8080
# ...response...
```

#### 2. Set up Port Forwarding
```bash
$ kubectl port-forward service/hello-minikube 8081:8080 &
# ...forwarding info...
```

#### 3. Test local connection
```bash
$ curl localhost:8081
# ...response...
```

#### 4. Stop Port Forwarding
```bash
$ ps
$ kill [kubectl_PID]
$ ps
```

---

## Resource Cleanup

#### 1. Delete Service
```bash
$ kubectl get services
$ kubectl delete service hello-minikube
$ kubectl get service
```

#### 2. Delete Deployment
```bash
$ kubectl get deployments
$ kubectl delete deployment hello-minikube
$ kubectl get deployments
$ kubectl get pods
```

#### 3. Stop and Delete Cluster (optional)
```bash
$ minikube status
$ minikube stop
$ minikube delete
$ minikube status
```

---

#### Cluster Management
- `minikube start` - Start cluster
- `minikube stop` - Stop cluster
- `minikube delete` - Delete cluster
- `minikube status` - Check cluster status

#### Pod Management
- `kubectl get pods` - List all pods
- `kubectl describe pod [pod_name]` - Pod details
- `kubectl logs [pod_name]` - Pod logs

#### Service Management
- `kubectl get services` - List all services
- `kubectl expose deployment [name] --port=[port]` - Create service
- `kubectl delete service [name]` - Delete service

#### Networking
- `kubectl port-forward service/[service_name] [local_port]:[service_port]` - Port forwarding

---

## Notes

1. **Permissions**: Ensure your user is in the docker group
2. **Network**: Stable internet is required to download Docker images
3. **Resources**: MiniKube needs enough memory and CPU
4. **Firewall**: Make sure required ports are open 