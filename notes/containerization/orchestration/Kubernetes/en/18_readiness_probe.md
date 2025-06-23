# Kubernetes Readiness Probe Implementation

[English](../en/18_readiness_probe.md) | [繁體中文](../zh-tw/18_readiness_probe.md) | [日本語](../ja/18_readiness_probe.md) | [Back to Index](../README.md)


## Overview
Readiness Probe is used to check if a Pod is ready to receive traffic. When the Readiness Probe fails, the Pod is removed from the Service's load balancer, but the Pod itself is not restarted.

## Prerequisites

### Start Docker Environment
```bash
$ sudo service docker start
$ docker ps
```

### Create Minikube Cluster
```bash
$ minikube start --driver docker
$ minikube status
```

## Create Deployment Configuration

### Create YAML File
```bash
$ vi advanced-deployment.yaml
```

### Deployment Configuration Content
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

### View Configuration File
```bash
$ cat advanced-deployment.yaml
```

## Deployment and Monitoring

### Deploy Deployment
```bash
$ kubectl apply -f advanced-deployment.yaml
```

### Monitor ReplicaSet
```bash
$ kubectl get rs -w
NAME                        DESIRED   CURRENT   READY   AGE
app-deployment-7f57cfb4b6   3         3         0       10s
app-deployment-7f57cfb4b6   3         3         1       15s
app-deployment-7f57cfb4b6   3         3         2       15s
app-deployment-7f57cfb4b6   3         3         3       15s
```

### Monitor Pod Status
```bash
$ kubectl get pods -w
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-7f57cfb4b6-g5sz7   1/1     Running   0          26s
app-deployment-7f57cfb4b6-mjnzw   1/1     Running   0          26s
app-deployment-7f57cfb4b6-q6xdp   1/1     Running   0          26s
app-deployment-7f57cfb4b6-q6xdp   0/1     Running   0          3m33s
```

## Create Service

### Deploy NodePort Service
```bash
$ kubectl apply -f simple-service-nodeport.yaml
```

### Get Node Information
```bash
$ kubectl get node
NAME       STATUS   ROLES           AGE    VERSION
minikube   Ready    control-plane   2d8h   v1.33.1

$ kubectl describe node minikube | grep InternalIP
InternalIP:  192.168.49.2
```

### Get Service Information
```bash
$ kubectl get services
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
app-service-nodeport   NodePort    10.111.160.67   <none>        8080:30080/TCP   93s
kubernetes             ClusterIP   10.96.0.1       <none>        443/TCP          22h
```

## Test Readiness Probe

### Simulate Health Check Failure
```bash
$ curl http://192.168.49.2:30080/healthcheck_dependency_switchstatus
[beta] served by: app-deployment-7f57cfb4b6-q6xdp.
[beta] isDependancyHealthy value switched to false.
```

### Monitor Pod Status Changes
In a new terminal window, run:
```bash
$ kubectl get pods -w
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-7f57cfb4b6-g5sz7   1/1     Running   0          26s
app-deployment-7f57cfb4b6-mjnzw   1/1     Running   0          26s
app-deployment-7f57cfb4b6-q6xdp   1/1     Running   0          26s
app-deployment-7f57cfb4b6-q6xdp   0/1     Running   0          3m33s
```
Expected result: You should see `Ready 0/1` status

### Test Application Access
```bash
# You won't see response from app-deployment-7f57cfb4b6-q6xdp
$ curl http://${NODE_IP}:${NODE_PORT}/
```

### Check Detailed Pod Status
```bash
$ kubectl describe pod [pod_id]
Events:
  Type     Reason     Age                 From               Message
  ----     ------     ----                ----               -------
  Normal   Scheduled  4m32s               default-scheduler  Successfully assigned default/app-deployment-7f57cfb4b6-q6xdp to minikube
  Normal   Pulling    4m32s               kubelet            Pulling image "uopsdod/k8s-hostname-amd64-beta:v1"
  Normal   Pulled     4m29s               kubelet            Successfully pulled image "uopsdod/k8s-hostname-amd64-beta:v1" in 1.456s (3.11s including waiting). Image size: 914594685 bytes.
  Normal   Created    4m29s               kubelet            Created container: app-container
  Normal   Started    4m28s               kubelet            Started container app-container
  Warning  Unhealthy  42s (x25 over 65s)  kubelet            Readiness probe failed: HTTP probe failed with statuscode: 500
```
Expected result: You should see `Readiness probe failed: HTTP probe failed with statuscode: 500`

## Clean Up Resources

### Delete All Deployed Resources
```bash
$ kubectl delete deployments --all
$ kubectl delete services --all
```

## Important Notes

- **Readiness Probe**: Checks if Pod is ready to receive traffic
- **Liveness Probe**: Checks if Pod is alive, restarts Pod when failed
- **Startup Probe**: Used during Pod startup period, switches to Liveness Probe after completion
- When Readiness Probe fails, Pod is removed from Service's load balancer but not restarted