# Kubernetes ReplicaSet Deployment Practice

[English](../en/09_deploy_replica_set.md) | [繁體中文](../zh-tw/09_deploy_replica_set.md) | [日本語](../ja/09_deploy_replica_set.md) | [Back to Index](../README.md)

## Prerequisites

#### Start Docker Service
```bash
$ sudo service docker start
$ docker ps
```

#### Create Minikube Cluster
```bash
$ minikube start --driver docker
$ minikube status
```

## Create ReplicaSet

#### Create ReplicaSet Definition File
Create `simple-replicaset.yaml` file:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: app-rs
spec: 
  replicas: 3
  selector: 
    matchLabels:
      app: app-pod
  template: 
    metadata: 
      labels:
        app: app-pod
    spec:
      containers:
      - name: app-container
        image: uopsdod/k8s-hostname-amd64-beta:v1
        ports:
        - containerPort: 80
```

#### Deploy ReplicaSet
```bash
# Deploy ReplicaSet
$ kubectl apply -f simple-replicaset.yaml

# Check ReplicaSet status
$ kubectl get rs
NAME     DESIRED   CURRENT   READY   AGE
app-rs   3         3         3       26s

# Check Pod status, wait for Pods to start
$ kubectl get pods
NAME           READY   STATUS    RESTARTS   AGE
app-rs-24r5n   1/1     Running   0          32s
app-rs-7fswq   1/1     Running   0          32s
app-rs-bhc7x   1/1     Running   0          32s
```

## Test ReplicaSet Self-Healing Mechanism

#### Simulate Pod Failure
```bash
# Delete all Pods to test ReplicaSet's automatic reconstruction
$ kubectl delete pods --all
pod "app-rs-24r5n" deleted
pod "app-rs-7fswq" deleted
pod "app-rs-bhc7x" deleted

# Observe ReplicaSet automatically creating new Pods
$ kubectl get pods
NAME           READY   STATUS    RESTARTS   AGE
app-rs-4mb4x   1/1     Running   0          35s
app-rs-rjdvn   1/1     Running   0          35s
app-rs-vkct7   1/1     Running   0          35s
```

## Clean Up Resources
```bash
# Delete all ReplicaSets
$ kubectl delete rs --all
```

## Explanation

- **ReplicaSet**: Ensures a specified number of Pod replicas are continuously running, providing high availability and load distribution
- **replicas: 3**: Specifies the number of Pod replicas to maintain
- **selector**: Defines how to identify Pods belonging to this ReplicaSet
- **template**: Defines the Pod specification, including container image, ports, and other settings
- **Self-healing**: When Pods are deleted or fail, ReplicaSet automatically creates new Pods to maintain the specified replica count
- **kubectl get rs**: View ReplicaSet status, showing desired replicas, current replicas, and other information

## Key Concepts

- **Desired State vs Actual State**: ReplicaSet continuously monitors and ensures the actual number of running Pods matches the desired replica count
- **Label Selector**: Uses `matchLabels` to identify and manage Pods belonging to this ReplicaSet
- **Pod Template**: Defines the specification and settings that newly created Pods should have 