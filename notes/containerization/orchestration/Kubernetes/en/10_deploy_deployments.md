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


## About Self-healing

- Pod Level:
  - Controllers like Deployment / ReplicaSet / StatefulSet maintain the desired replica count. If a Pod crashes or gets deleted, the controller will automatically create new Pods to fulfill the requirement. This is what K8S commonly refers to as "heal pods".

- Node Level:
  - Kubernetes itself does not repair or rebuild Nodes. If a Node fails (hardware failure, network disconnection, or VM deletion), K8S will only mark this Node as NotReady, and after a certain period (determined by node-controller), mark the Pods on that Node as failed, then reschedule new Pods on other available Nodes.

> ⚠️ In other words, K8S can only achieve "pod rescheduling", but cannot "heal node" or make nodes revive themselves.

#### How to heal Nodes?

- This usually relies on external infrastructure layers, for example:
  - Cloud environments:
    - AWS EKS → Combined with Auto Scaling Group (ASG) + Cluster Autoscaler, automatically replace failed EC2 nodes.
    - GKE / AKS → Built-in node auto-repair functionality.
  - On-premise environments:
    - May use MetalLB + Cluster Autoscaler or additional monitoring/automation tools (such as Ansible, Terraform, or hardware management systems).

- Summary:
  - K8S heals Pods, but Node "healing" depends on cloud platforms or additional node management tools.

## About Rolling Update
Deployment rolling update process

1. User executes command (kubectl apply -f ...)
  - Updates Deployment specifications (e.g., image: v2).
  - This new desired state is written to etcd (K8S's single source of truth storage).
2. Deployment Controller detects differences
  - It reads from etcd that the "desired state" differs from "actual state".
  - For example: currently nginx:v1 ReplicaSet has 5 Pods, desired is nginx:v2.
3. Generate new ReplicaSet
  - Controller creates a new ReplicaSet (corresponding to nginx:v2).
  - The old ReplicaSet (nginx:v1) won't be deleted immediately, but gradually scaled down.
4. Rolling Update (RollingUpdate)
  - Controller controls according to Deployment strategy (spec.strategy.rollingUpdate):
    - maxUnavailable: How many Pods can be unavailable at once.
    - maxSurge: Maximum number of additional Pods that can be created at once.
  - Actually "simultaneously scale up new RS, scale down old RS" until reaching target numbers.
5. State Recording
  - Each Deployment's status (status.conditions, status.replicas, status.updatedReplicas ...) is updated and stored in etcd.
  - This allows kubectl rollout status deployment/... to check current update progress.

Summary:
  - etcd is the single source of truth, recording all Deployment specs and update progress in etcd.
  - Deployment Controller is the executor, responsible for Pod replacement based on "desired state" in etcd, and writing process status back to etcd. 