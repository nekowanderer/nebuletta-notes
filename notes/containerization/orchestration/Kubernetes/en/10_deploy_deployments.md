# Kubernetes Deployment Deployment Practice

[English](../en/10_deploy_deployments.md) | [繁體中文](../zh-tw/10_deploy_deployments.md) | [日本語](../ja/10_deploy_deployments.md) | [Back to Index](../README.md)

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

## Create Deployment

#### Create Deployment Definition File
Create `simple-deployment.yaml` file:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
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

#### Deploy Deployment
```bash
# Deploy Deployment
$ kubectl apply -f simple-deployment.yaml

# Check Deployment status
$ kubectl get deployments
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
app-deployment   3/3     3            3           19s

# Check ReplicaSet status
$ kubectl get rs
NAME                        DESIRED   CURRENT   READY   AGE
app-deployment-8687d78656   3         3         3       28s

# Check Pod status
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-8687d78656-5bf82   1/1     Running   0          31s
app-deployment-8687d78656-l4ctt   1/1     Running   0          31s
app-deployment-8687d78656-lv7wl   1/1     Running   0          31s

# Check all resources
$ kubectl get all
NAME                                  READY   STATUS    RESTARTS   AGE
pod/app-deployment-8687d78656-5bf82   1/1     Running   0          37s
pod/app-deployment-8687d78656-l4ctt   1/1     Running   0          37s
pod/app-deployment-8687d78656-lv7wl   1/1     Running   0          37s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   51m

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/app-deployment   3/3     3            3           37s

NAME                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/app-deployment-8687d78656   3         3         3       37s
```

## Test Deployment State Maintenance Mechanism

#### Simulate Pod Failure
```bash
# Delete all Pods to test Deployment's automatic reconstruction capability
$ kubectl delete pods --all
pod "app-deployment-8687d78656-5bf82" deleted
pod "app-deployment-8687d78656-l4ctt" deleted
pod "app-deployment-8687d78656-lv7wl" deleted

# Observe Deployment automatically creating new Pods
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-8687d78656-2hhtc   1/1     Running   0          37s
app-deployment-8687d78656-4hcvv   1/1     Running   0          37s
app-deployment-8687d78656-4xfcd   1/1     Running   0          37s
```

#### Test ReplicaSet Deletion
```bash
# Delete all ReplicaSets
$ kubectl delete rs --all
replicaset.apps "app-deployment-8687d78656" deleted

# Observe Deployment automatically creating new ReplicaSet
$ kubectl get rs
NAME                        DESIRED   CURRENT   READY   AGE
app-deployment-8687d78656   3         3         3       32s

# Confirm Pods still exist
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-8687d78656-6hzmm   1/1     Running   0          78s
app-deployment-8687d78656-865xb   1/1     Running   0          78s
app-deployment-8687d78656-qm75q   1/1     Running   0          78s
```

## Clean Up Resources
```bash
# Delete all Deployments
$ kubectl delete deployments --all
```

## Explanation

- **Deployment**: Provides declarative updates for Pods and ReplicaSets, the most commonly used workload resource in Kubernetes
- **replicas: 3**: Specifies the number of Pod replicas to maintain
- **selector**: Defines how to identify Pods belonging to this Deployment
- **template**: Defines Pod specifications, including container image, ports, and other settings
- **Self-healing**: When Pods are deleted or fail, Deployment automatically creates new Pods to maintain the specified replica count
- **kubectl get deployments**: Check Deployment status, showing ready replicas, up-to-date replicas, and other information

## Key Concepts

- **Deployment vs ReplicaSet**: Deployment manages ReplicaSets, while ReplicaSets manage Pods, forming a hierarchical management structure
- **Desired State vs Actual State**: Deployment continuously monitors and ensures the actual number of running Pods matches the desired replica count
- **Label Selector**: Uses `matchLabels` to identify and manage Pods belonging to this Deployment
- **Pod Template**: Defines the specifications and settings that newly created Pods should have
- **Version Control**: Deployment supports rolling updates and version rollback functionality (advanced features)
- **Resource Hierarchy**: Deployment > ReplicaSet > Pod, deleting upper-level resources affects lower-level resources 