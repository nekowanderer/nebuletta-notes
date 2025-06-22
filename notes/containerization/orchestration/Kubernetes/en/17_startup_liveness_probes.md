# Kubernetes Startup and Liveness Probe Implementation

[English](../en/17_startup_liveness_probes.md) | [繁體中文](../zh-tw/17_startup_liveness_probes.md) | [日本語](../ja/17_startup_liveness_probes.md) | [Back to Index](../README.md)

This document demonstrates how to implement Startup Probe and Liveness Probe in Kubernetes, and observe probe behavior through practical operations.

### Prerequisites

##### 1. Start Docker Environment
```bash
$ sudo service docker start
$ docker ps
```

##### 2. Create Minikube Cluster
```bash
$ minikube start --driver docker
$ minikube status
```

### Create Deployment Configuration

Reuse the simple-deployment.yml from [10_deploy_deployments](./10_deploy_deployments.md)

##### 1. Copy Base Configuration
```bash
$ cp simple-deployment.yaml advanced-deployment.yaml
```

##### 2. Edit Configuration File
Use `vi advanced-deployment.yaml` to edit the file, adding Startup Probe and Liveness Probe configurations:

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
        
        # Startup Probe: Give application startup time
        startupProbe:
          httpGet:
            path: /healthcheck
            port: 80
          initialDelaySeconds: 0
          timeoutSeconds: 1
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 30
        
        # Liveness Probe: Monitor application liveness
        livenessProbe:
          httpGet:
            path: /healthcheck
            port: 80
          initialDelaySeconds: 0
          timeoutSeconds: 1
          periodSeconds: 1
          successThreshold: 1
          failureThreshold: 5
```

##### 3. Verify Configuration
```bash
$ cat advanced-deployment.yaml
```

### Deployment and Monitoring

##### 1. Deploy Deployment
```bash
$ kubectl apply -f advanced-deployment.yaml
```

##### 2. Monitor ReplicaSet
```bash
$ kubectl get rs -w
NAME                        DESIRED   CURRENT   READY   AGE
app-deployment-6786b5648d   3         3         0       8s
app-deployment-6786b5648d   3         3         1       11s
app-deployment-6786b5648d   3         3         2       11s
app-deployment-6786b5648d   3         3         3       11s
```

##### 3. Deploy Service
```bash
$ kubectl apply -f simple-service-nodeport.yaml
```

### Test Probe Functionality

##### 1. Get Node Information
```bash
$ kubectl get node
$ kubectl describe node minikube
...
Addresses:
  InternalIP:  192.168.49.2
  Hostname:    minikube
...

# Set node IP (adjust according to actual output)
$ NODE_IP=192.168.49.2
```

##### 2. Get Service Information
```bash
$ kubectl get services
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
app-service-nodeport   NodePort    10.104.200.155   <none>        8080:30080/TCP   54s
kubernetes             ClusterIP   10.96.0.1        <none>        443/TCP          7h54m

# Set NodePort (adjust according to actual output)
$ NODE_PORT=3XXXX
```

##### 3. Simulate Health Check Failure
```bash
$ curl http://${NODE_IP}:${NODE_PORT}/healthcheck_switchstatus
[beta] served by: app-deployment-6786b5648d-757bq.
[beta] isHealthy value switched to false.
```

##### 4. Monitor Pod Status
```bash
$ kubectl get pods -w
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-6786b5648d-757bq   1/1     Running   0          3m42s
app-deployment-6786b5648d-jlvdw   1/1     Running   0          3m42s
app-deployment-6786b5648d-psqph   1/1     Running   0          3m42s
app-deployment-6786b5648d-757bq   0/1     Running   1 (2s ago)   4m9s
app-deployment-6786b5648d-757bq   0/1     Running   1 (3s ago)   4m10s
app-deployment-6786b5648d-757bq   1/1     Running   1 (3s ago)   4m10s

# View detailed information of specific Pod
$ kubectl describe pod [pod_id]
...
Events:
  Type     Reason     Age                  From               Message
  ----     ------     ----                 ----               -------
  Normal   Scheduled  4m36s                default-scheduler  Successfully assigned default/app-deployment-6786b5648d-757bq to minikube
  Normal   Pulled     4m35s                kubelet            Successfully pulled image "uopsdod/k8s-hostname-amd64-beta:v1" in 1.663s (1.663s including waiting). Image size: 914594685 bytes.
  Warning  Unhealthy  60s (x5 over 64s)    kubelet            Liveness probe failed: HTTP probe failed with statuscode: 500
  Normal   Killing    60s                  kubelet            Container app-container failed liveness probe, will be restarted
  Normal   Pulling    30s (x2 over 4m36s)  kubelet            Pulling image "uopsdod/k8s-hostname-amd64-beta:v1"
  Normal   Created    28s (x2 over 4m35s)  kubelet            Created container: app-container
  Normal   Started    28s (x2 over 4m35s)  kubelet            Started container app-container
  Normal   Pulled     28s                  kubelet            Successfully pulled image "uopsdod/k8s-hostname-amd64-beta:v1" in 1.462s (1.462s including waiting). Image size: 914594685 bytes.
...
```

### Clean Up Resources

```bash
$ kubectl delete deployments --all
$ kubectl delete services --all
```

### Probe Configuration Explanation

##### Startup Probe Parameters
- **`initialDelaySeconds: 0`**: Start probing immediately after container startup
- **`timeoutSeconds: 1`**: Timeout for each probe is 1 second
- **`periodSeconds: 10`**: Execute probe every 10 seconds
- **`successThreshold: 1`**: Consider startup complete after 1 success
- **`failureThreshold: 30`**: Allow up to 30 failures (total 5 minutes startup time)

##### Liveness Probe Parameters
- **`initialDelaySeconds: 0`**: Start probing immediately after container startup
- **`timeoutSeconds: 1`**: Timeout for each probe is 1 second
- **`periodSeconds: 1`**: Execute probe every 1 second
- **`successThreshold: 1`**: Consider alive after 1 success
- **`failureThreshold: 5`**: Allow up to 5 failures (restart container after 5 seconds)

### Expected Results

1. **Startup Phase**: Startup Probe will give the application sufficient startup time
2. **Running Phase**: Liveness Probe will continuously monitor application status
3. **Failure Handling**: When health check fails, Kubernetes will automatically restart the container
4. **Monitoring Verification**: Detailed probe execution records can be viewed through `kubectl describe pod`