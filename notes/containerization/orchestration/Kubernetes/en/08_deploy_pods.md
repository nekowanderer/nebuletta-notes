# Kubernetes Pod Deployment Practice

[English](../en/08_deploy_pods.md) | [繁體中文](../zh-tw/08_deploy_pods.md) | [日本語](../ja/08_deploy_pods.md) | [Back to Index](../README.md)

## Prerequisites

#### Start Docker Service
```bash
sudo service docker start
docker ps
```

#### Create Minikube Cluster
```bash
minikube start --driver docker
minikube status
```

## Create Pod

#### Create Pod Definition File
Create `simple-pod.yaml` file:

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

#### Deploy Pod
```bash
# Deploy Pod
kubectl apply -f simple-pod.yaml

# Check Pod status
kubectl get pods

# Monitor Pod status changes in real-time
kubectl get pods -w
```

## Clean Up Resources
```bash
# Delete all Pods
kubectl delete pods --all
```

## Explanation

- **Pod**: The smallest deployment unit in Kubernetes, can contain one or more containers
- **kubectl apply**: Create or update Kubernetes resources
- **kubectl get pods -w**: Or `--watch`, monitor Pod status in real-time, press Ctrl+C to stop monitoring
- **kubectl delete pods --all**: Delete all Pods for cleanup 