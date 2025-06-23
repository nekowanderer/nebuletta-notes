# Kubernetes Rolling Updates Implementation

[English](../en/19_k8s_rolling_updates.md) | [繁體中文](../zh-tw/19_k8s_rolling_updates.md) | [日本語](../ja/19_k8s_rolling_updates.md) | [Back to Index](../README.md)

## Prerequisites

### Start Docker Service
```bash
$ sudo service docker start
$ docker ps
```

### Create Minikube Cluster
```bash
$ minikube start --driver docker
$ minikube status
```

## Deploy Initial Version (v1)

### Create Deployment Definition File
Copy and create the rolling update deployment file:

```bash
$ cp advanced-deployment.yaml advanced-deployment-rollingupdate.yaml
$ cat advanced-deployment-rollingupdate.yaml
```

### Deploy v1 Version
```bash
$ kubectl apply -f advanced-deployment-rollingupdate.yaml

$ kubectl get deployments
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
app-deployment   3/3     3            3           22s

$ kubectl get rs
NAME                        DESIRED   CURRENT   READY   AGE
app-deployment-7f57cfb4b6   3         3         3       24s
```

### Confirm Pod Status
```bash
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-7f57cfb4b6-5nmhg   1/1     Running   0          48s
app-deployment-7f57cfb4b6-qns74   1/1     Running   0          48s
app-deployment-7f57cfb4b6-stpv9   1/1     Running   0          48s

$ kubectl describe pod [pod_id] | grep Image:
    Image:          uopsdod/k8s-hostname-amd64-beta:v1
```

## Configure Rolling Update Strategy

### Modify Deployment Configuration
Edit the `advanced-deployment-rollingupdate.yaml` file to add rolling update strategy and update the image version:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
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
        image: uopsdod/k8s-hostname-amd64-beta:v2
        imagePullPolicy: Always
        ports:
        - containerPort: 80
        startupProbe:
          httpGet:
            path: /healthcheck
            port: 80
          initialDelaySeconds: 0
          timeoutSeconds: 1
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 30
        livenessProbe:
          httpGet:
            path: /healthcheck
            port: 80
          initialDelaySeconds: 0
          timeoutSeconds: 1
          periodSeconds: 1
          successThreshold: 1
          failureThreshold: 5
        readinessProbe:
          httpGet:
            path: /healthcheck_dependency
            port: 80
          initialDelaySeconds: 0
          timeoutSeconds: 1
          periodSeconds: 1
          successThreshold: 5
          failureThreshold: 7
```

### Verify Configuration
```bash
$ cat advanced-deployment-rollingupdate.yaml
```

## Execute Rolling Update

### Deploy v2 Version
```bash
$ kubectl apply -f advanced-deployment-rollingupdate.yaml
```

### Monitor Rolling Update Process
Run the following commands in a new terminal window to monitor ReplicaSet changes in real-time:

```bash
$ kubectl get rs -w
NAME                        DESIRED   CURRENT   READY   AGE
app-deployment-7f57cfb4b6   3         3         3       3m12s
app-deployment-6f975d88d9   1         0         0       0s
app-deployment-6f975d88d9   1         0         0       0s
app-deployment-6f975d88d9   1         1         0       0s
app-deployment-6f975d88d9   1         1         1       14s
app-deployment-7f57cfb4b6   2         3         3       4m41s
app-deployment-6f975d88d9   2         1         1       14s
app-deployment-7f57cfb4b6   2         3         3       4m41s
app-deployment-6f975d88d9   2         1         1       14s
app-deployment-6f975d88d9   2         2         1       14s
app-deployment-7f57cfb4b6   2         2         2       4m41s
app-deployment-6f975d88d9   2         2         2       29s
app-deployment-7f57cfb4b6   1         2         2       4m56s
app-deployment-7f57cfb4b6   1         2         2       4m56s
app-deployment-6f975d88d9   3         2         2       29s
app-deployment-7f57cfb4b6   1         1         1       4m56s
app-deployment-6f975d88d9   3         2         2       29s
app-deployment-6f975d88d9   3         3         2       29s
app-deployment-6f975d88d9   3         3         3       44s
app-deployment-7f57cfb4b6   0         1         1       5m11s
app-deployment-7f57cfb4b6   0         1         1       5m11s
app-deployment-7f57cfb4b6   0         0         0       5m11s

$ kubectl get pods -w
NAME                              READY   STATUS              RESTARTS   AGE
app-deployment-6f975d88d9-t74gf   0/1     ContainerCreating   0          5s
app-deployment-7f57cfb4b6-5nmhg   1/1     Running             0          4m32s
app-deployment-7f57cfb4b6-qns74   1/1     Running             0          4m32s
app-deployment-7f57cfb4b6-stpv9   1/1     Running             0          4m32s
app-deployment-6f975d88d9-t74gf   0/1     Running             0          10s
app-deployment-6f975d88d9-t74gf   0/1     Running             0          11s
app-deployment-6f975d88d9-t74gf   1/1     Running             0          14s
app-deployment-7f57cfb4b6-stpv9   1/1     Terminating         0          4m41s
app-deployment-6f975d88d9-cq2xk   0/1     Pending             0          0s
app-deployment-6f975d88d9-cq2xk   0/1     Pending             0          0s
app-deployment-6f975d88d9-cq2xk   0/1     ContainerCreating   0          0s
app-deployment-6f975d88d9-cq2xk   0/1     Running             0          3s
app-deployment-6f975d88d9-cq2xk   0/1     Running             0          10s
app-deployment-6f975d88d9-cq2xk   1/1     Running             0          15s
app-deployment-7f57cfb4b6-qns74   1/1     Terminating         0          4m56s
app-deployment-6f975d88d9-2lwcb   0/1     Pending             0          0s
app-deployment-6f975d88d9-2lwcb   0/1     Pending             0          0s
app-deployment-6f975d88d9-2lwcb   0/1     ContainerCreating   0          0s
app-deployment-6f975d88d9-2lwcb   0/1     Running             0          3s
app-deployment-6f975d88d9-2lwcb   0/1     Running             0          10s
app-deployment-7f57cfb4b6-stpv9   0/1     Error               0          5m11s
app-deployment-6f975d88d9-2lwcb   1/1     Running             0          15s
app-deployment-7f57cfb4b6-5nmhg   1/1     Terminating         0          5m11s
app-deployment-7f57cfb4b6-stpv9   0/1     Error               0          5m11s
app-deployment-7f57cfb4b6-stpv9   0/1     Error               0          5m11s
app-deployment-7f57cfb4b6-qns74   0/1     Error               0          5m27s
app-deployment-7f57cfb4b6-5nmhg   0/1     Error               0          5m42s
```

### Verify Update Results
```bash
# Check ReplicaSet status
$ kubectl get rs
NAME                        DESIRED   CURRENT   READY   AGE
app-deployment-6f975d88d9   3         3         3       114s
app-deployment-7f57cfb4b6   0         0         0       6m21s

# Check Pod status
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-6f975d88d9-2lwcb   1/1     Running   0          81s
app-deployment-6f975d88d9-cq2xk   1/1     Running   0          96s
app-deployment-6f975d88d9-t74gf   1/1     Running   0          110s

# Verify image version used by Pods
$ kubectl describe pod [pod_id] | grep Image:
    Image:          uopsdod/k8s-hostname-amd64-beta:v2
```

## Rolling Update Strategy Explanation

### Strategy Parameters
- **type: RollingUpdate**: Specifies the use of rolling update strategy
- **maxSurge: 1**: Maximum number allowed above the desired replica count (number of Pods that can run simultaneously)
- **maxUnavailable: 0**: Number of unavailable Pods during update (0 means no service interruption allowed)

### Update Process
1. Kubernetes creates a new ReplicaSet to deploy the v2 version
2. Gradually replaces old version Pods with new version
3. Ensures service remains available during the update process

## Important Concepts

- **Rolling Update**: A strategy to gradually update Pods ensuring no service interruption
- **maxSurge**: Controls the number that can exceed the desired replica count during update
- **maxUnavailable**: Controls the number of Pods allowed to be unavailable during update
- **ReplicaSet Management**: Each version generates a new ReplicaSet to manage corresponding Pods
- **Health Checks**: Uses probes to ensure new Pods are working properly before proceeding to the next update step
