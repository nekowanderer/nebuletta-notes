# Kubernetes Namespace Practical Operations

[English](../en/25_k8s_namespace_in_practice.md) | [繁體中文](../zh-tw/25_k8s_namespace_in_practice.md) | [日本語](../ja/25_k8s_namespace_in_practice.md) | [Back to Index](../README.md)

## Prerequisites

### Starting Docker Environment
```bash
# Start Docker service
$ sudo service docker start

# Check Docker status
$ docker ps
```

### Setting up Minikube Cluster
```bash
# Start minikube with Docker driver
$ minikube start --driver docker

# Check cluster status
$ minikube status
```

## Creating Namespace and Applications

### Creating YAML Configuration File
Create `beta-app-all.yaml` file:

```yaml
# Create namespace
apiVersion: v1
kind: Namespace
metadata:
  name: app-ns
---
# Create Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: beta-app-deployment
  namespace: app-ns
spec:
  replicas: 3
  selector:
    matchLabels:
      app: beta-app-pod
  template:
    metadata:
      labels:
        app: beta-app-pod
    spec:
      containers:
      - name: beta-app-container
        image: uopsdod/k8s-hostname-amd64-beta:v1
        ports: 
        - containerPort: 80
---
# Create Service
apiVersion: v1
kind: Service
metadata:
  name: beta-app-service-clusterip
  namespace: app-ns
spec:
  selector:
    app: beta-app-pod
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
```
Here, `---` represents deploying multiple resource types within the same yaml file.

## Deployment and Verification

### Deploying Applications
```bash
# Deploy all resources
$ kubectl apply -f beta-app-all.yaml
```

### Checking Deployment Status
```bash
# View all namespaces
$ kubectl get ns
NAME              STATUS   AGE
app-ns            Active   7s
default           Active   7d9h
kube-node-lease   Active   7d9h
kube-public       Active   7d9h
kube-system       Active   7d9h

# View all resources (default namespace)
$ kubectl get all
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   5d

# View all resources in a specific namespace
$ kubectl get all --namespace app-ns
NAME                                       READY   STATUS    RESTARTS   AGE
pod/beta-app-deployment-865c646d9d-9ml47   1/1     Running   0          54s
pod/beta-app-deployment-865c646d9d-shc9b   1/1     Running   0          54s
pod/beta-app-deployment-865c646d9d-xl8v6   1/1     Running   0          54s

NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/beta-app-service-clusterip   ClusterIP   10.103.190.75   <none>        8080/TCP   54s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/beta-app-deployment   3/3     3            3           54s

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/beta-app-deployment-865c646d9d   3         3         3       54s

# Or use shorthand syntax
$ kubectl get all -n app-ns
```

## Resource Cleanup

### Deleting Namespace
```bash
# Delete entire namespace (will also delete all resources within that namespace)
$ kubectl delete ns app-ns
namespace "app-ns" deleted

# Confirm namespace has been deleted
$ kubectl get ns
NAME              STATUS   AGE
default           Active   7d9h
kube-node-lease   Active   7d9h
kube-public       Active   7d9h
kube-system       Active   7d9h

# Try to view resources in the deleted namespace (should show an error)
$ kubectl get all -n app-ns
No resources found in app-ns namespace.
```

## Important Concepts

- **Namespace Isolation**: Resources in different namespaces are mutually isolated
- **Resource Ownership**: Both Deployment and Service belong to the specified namespace
- **Deletion Behavior**: Deleting a namespace will simultaneously delete all resources within that namespace
- **Cross-Namespace Access**: By default, resources in different namespaces cannot directly access each other
