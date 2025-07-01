# Kubernetes HPA (Horizontal Pod Autoscaler) Implementation

[English](../en/33_k8s_hpa_in_practice.md) | [繁體中文](../zh-tw/33_k8s_hpa_in_practice.md) | [日本語](../ja/33_k8s_hpa_in_practice.md) | [Back to Index](../README.md)

## Overview
This guide demonstrates how to set up and use Horizontal Pod Autoscaler (HPA) in Kubernetes to achieve automatic scaling based on CPU usage.

## Prerequisites

### 1. Start Docker Environment
```bash
$ sudo service docker start
$ docker ps
```

### 2. Create Minikube Cluster
```bash
$ minikube start --driver docker
$ minikube status
```

### 3. Enable Ingress Add-on
```bash
$ minikube addons enable ingress

$ kubectl get all -n ingress-nginx
NAME                                           READY   STATUS      RESTARTS      AGE
pod/ingress-nginx-admission-create-5fj8c       0/1     Completed   0             46h
pod/ingress-nginx-admission-patch-529hj        0/1     Completed   1             46h
pod/ingress-nginx-controller-67c5cb88f-dv4px   1/1     Running     2 (48m ago)   46h

NAME                                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx-controller             NodePort    10.102.245.181   <none>        80:30447/TCP,443:31545/TCP   46h
service/ingress-nginx-controller-admission   ClusterIP   10.96.249.142    <none>        443/TCP                      46h

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-nginx-controller   1/1     1            1           46h

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-nginx-controller-67c5cb88f   1         1         1       46h

NAME                                       STATUS     COMPLETIONS   DURATION   AGE
job.batch/ingress-nginx-admission-create   Complete   1/1           7s         46h
job.batch/ingress-nginx-admission-patch    Complete   1/1           7s         46h
```

## Deploy Application

### 1. Create HPA Version Application Configuration
Copy and modify existing application configuration file:

```bash
$ cp beta-app-all.yaml beta-app-all-hpa.yaml
$ vi beta-app-all-hpa.yaml
```

### 2. Application Configuration File Content
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app-ns
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: beta-app-deployment
  namespace: app-ns
spec:
  replicas: 1  # Change to 1
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
        resources:     # Add this section
          limits:
            cpu: 300m
          requests:
            cpu: 200m
---
apiVersion: v1
kind: Service
metadata:
  name: beta-app-service-clusterip
  namespace: app-ns
spec:
  type: ClusterIP      # Add this
  selector:
    app: beta-app-pod
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
```

**Important Changes:**
- Set `replicas: 1` as initial replica count
- Add `resources` section with CPU limits and requests
- CPU limit set to `300m`, request set to `200m`

### 3. Deploy Application
```bash
$ kubectl apply -f beta-app-all-hpa.yaml
namespace/app-ns created
deployment.apps/beta-app-deployment created
service/beta-app-service-clusterip created

$ kubectl get all -n app-ns
NAME                                       READY   STATUS    RESTARTS   AGE
pod/beta-app-deployment-8679d5777b-5kzjj   1/1     Running   0          9s

NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/beta-app-service-clusterip   ClusterIP   10.105.44.132   <none>        8080/TCP   9s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/beta-app-deployment   1/1     1            1           9s

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/beta-app-deployment-8679d5777b   1         1         1       9s
```

### 4. Deploy Production Application
```bash
$ kubectl apply -f prod-app-all.yaml
namespace/app-ns unchanged
deployment.apps/prod-app-deployment created
service/prod-app-service-clusterip created

$ kubectl get all -n app-ns
NAME                                       READY   STATUS    RESTARTS   AGE
pod/beta-app-deployment-8679d5777b-5kzjj   1/1     Running   0          36s
pod/prod-app-deployment-586c5dcc59-b88dw   1/1     Running   0          10s
pod/prod-app-deployment-586c5dcc59-fc82v   1/1     Running   0          10s
pod/prod-app-deployment-586c5dcc59-fdtdq   1/1     Running   0          10s

NAME                                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/beta-app-service-clusterip   ClusterIP   10.105.44.132    <none>        8080/TCP   36s
service/prod-app-service-clusterip   ClusterIP   10.100.206.114   <none>        8080/TCP   10s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/beta-app-deployment   1/1     1            1           36s
deployment.apps/prod-app-deployment   3/3     3            3           10s

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/beta-app-deployment-8679d5777b   1         1         1       36s
replicaset.apps/prod-app-deployment-586c5dcc59   3         3         3       10s
```

### 5. Deploy Ingress Routing
```bash
$ kubectl apply -f ingress-path.yaml

$ kubectl get ingress -n app-ns

$ kubectl get ingress -n app-ns -w
NAME           CLASS   HOSTS          ADDRESS   PORTS   AGE
ingress-path   nginx   all.demo.com             80      19s
ingress-path   nginx   all.demo.com   192.168.49.2   80      33s
```

## Configure Metrics Server

### 1. Download Metrics Server Configuration
```bash
$ wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
$ mv components.yaml metrics-server.yaml
```

### 2. Modify Metrics Server Configuration
```bash
$ vi metrics-server.yaml
```

Add `--kubelet-insecure-tls` parameter to container arguments:

```yaml
spec:
  containers:
  - args:
    - --cert-dir=/tmp
    - --secure-port=10250
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --kubelet-use-node-status-port
    - --metric-resolution=15s
    - --kubelet-insecure-tls  # Add this line
    image: k8s.gcr.io/metrics-server/metrics-server:v0.7.2
```

### 3. Deploy Metrics Server
```bash
$ kubectl apply -f metrics-server.yaml
$ kubectl get deployment metrics-server -n kube-system
$ kubectl get deployment metrics-server -n kube-system -w
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
metrics-server   0/1     1            0           19m
metrics-server   1/1     1            1           19m

# If it doesn't start, check logs with:
$ kubectl logs -n kube-system deployment/metrics-server
```

## Configure HPA

### 1. Create HPA
```bash
$ kubectl autoscale deployment beta-app-deployment --cpu-percent=10 --min=1 --max=10 -n app-ns
horizontalpodautoscaler.autoscaling/beta-app-deployment autoscaled

$ kubectl get hpa -n app-ns

$ kubectl get hpa -n app-ns -w
NAME                  REFERENCE                        TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%   1         10        1          78s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%   1         10        1          2m45s
```

**HPA Parameter Explanation:**
- `--cpu-percent=10`: Triggers scaling when CPU usage exceeds 10% of pod CPU requests. In this example:
  - HPA threshold = CPU requests × HPA percentage = 200 × 0.1 = 20m
- `--min=1`: Minimum replica count is 1
- `--max=10`: Maximum replica count is 10
- For more parameter details, see the `CPU Resource Configuration` section below

## Test Auto-scaling

### 1. Test Scale Up
Open a new terminal window:

```bash
$ kubectl get ingress -n app-ns
$ ingress_ip=192.168.49.2
$ curl ${ingress_ip}:80/beta -H 'Host: all.demo.com'
$ while sleep 0.005; do curl -s ${ingress_ip}:80/beta -H 'Host: all.demo.com'; done
```

This command continuously sends requests to the application. When CPU usage exceeds 10%, HPA will automatically increase Pod replicas.

### 2. Test Scale Down
Stop load testing:
```bash
# Press Ctrl + C to stop continuous requests
```

After stopping the load, HPA monitors CPU usage and automatically decreases Pod replicas when usage drops.

## Monitoring and Verification

### View HPA Status
```bash
$ kubectl describe hpa beta-app-deployment -n app-ns
Name:                                                  beta-app-deployment
Namespace:                                             app-ns
Labels:                                                <none>
Annotations:                                           <none>
CreationTimestamp:                                     Tue, 01 Jul 2025 14:07:28 +0000
Reference:                                             Deployment/beta-app-deployment
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  9% (19m) / 10%
Min replicas:                                          1
Max replicas:                                          10
Deployment pods:                                       3 current / 3 desired
Conditions:
  Type            Status  Reason              Message
  ----            ------  ------              -------
  AbleToScale     True    ReadyForNewScale    recommended size matches current size
  ScalingActive   True    ValidMetricFound    the HPA was able to successfully calculate a replica count from cpu resource utilization (percentage of request)
  ScalingLimited  False   DesiredWithinRange  the desired count is within the acceptable range
Events:
  Type    Reason             Age   From                       Message
  ----    ------             ----  ----                       -------
  Normal  SuccessfulRescale  62s   horizontal-pod-autoscaler  New size: 3; reason: cpu resource utilization (percentage of request) above target

# About five minutes after stopping curl loop:
$ kubectl get hpa -n app-ns
NAME                  REFERENCE                        TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%   1         10        1          78s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%   1         10        1          2m45s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%   1         10        1          3m
beta-app-deployment   Deployment/beta-app-deployment   cpu: 24%/10%   1         10        1          4m30s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 14%/10%   1         10        3          4m45s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 10%/10%   1         10        3          5m
beta-app-deployment   Deployment/beta-app-deployment   cpu: 9%/10%    1         10        3          5m15s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 8%/10%    1         10        3          5m45s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%    1         10        3          6m
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%    1         10        3          6m16s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%    1         10        3          6m31s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%    1         10        3          8m46s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%    1         10        3          10m
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%    1         10        1          11m


$ kubectl describe hpa beta-app-deployment -n app-ns
Name:                                                  beta-app-deployment
Namespace:                                             app-ns
Labels:                                                <none>
Annotations:                                           <none>
CreationTimestamp:                                     Tue, 01 Jul 2025 14:07:28 +0000
Reference:                                             Deployment/beta-app-deployment
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  0% (0) / 10%
Min replicas:                                          1
Max replicas:                                          10
Deployment pods:                                       3 current / 1 desired
Conditions:
  Type            Status  Reason            Message
  ----            ------  ------            -------
  AbleToScale     True    SucceededRescale  the HPA controller was able to update the target scale to 1
  ScalingActive   True    ValidMetricFound  the HPA was able to successfully calculate a replica count from cpu resource utilization (percentage of request)
  ScalingLimited  True    TooFewReplicas    the desired replica count is less than the minimum replica count
Events:
  Type    Reason             Age    From                       Message
  ----    ------             ----   ----                       -------
  Normal  SuccessfulRescale  6m17s  horizontal-pod-autoscaler  New size: 3; reason: cpu resource utilization (percentage of request) above target
  Normal  SuccessfulRescale  1s     horizontal-pod-autoscaler  New size: 1; reason: All metrics below target
```

## Resource Cleanup

### Clean up All Resources
```bash
$ kubectl delete namespace app-ns
```

## HPA Creation Methods

### HPA Creation Method Used in This Lab
This lab uses the `kubectl autoscale` command to create HPA, which uses Kubernetes [default behavior](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#default-behavior) settings:
- **Scale Down Stabilization**: 5 minutes (`stabilizationWindowSeconds: 300`)
- **Scale Up Stabilization**: Immediate (`stabilizationWindowSeconds: 0`)

### Custom HPA Configuration
If you need to adjust HPA behavior, you can create a custom HPA YAML file:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: beta-app-deployment
  namespace: app-ns
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: beta-app-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 10
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 600  # Custom scale-down stabilization time
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 180  # Custom scale-up stabilization time
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
```

### Modifying Existing HPA
You can also use the patch command to modify existing HPA:

```bash
$ kubectl patch hpa beta-app-deployment -n app-ns --type='merge' -p='
spec:
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 60
'
```

## CPU Resource Configuration

#### CPU Unit Explanation
- **1 CPU = 1000m (millicores)**
- **300m = 0.3 CPU = 30% of a single CPU core**
- **200m = 0.2 CPU = 20% of a single CPU core**

#### CPU Requests - 200m
```yaml
resources:
  requests:
    cpu: 200m
```

**Meaning:**
- **Resource Guarantee**: Kubernetes guarantees at least 200m CPU resources for the Pod
- **Scheduling Basis**: Scheduler only places Pods on nodes with sufficient CPU resources
- **HPA Calculation Base**: HPA uses the requests value as the base for CPU usage percentage calculation
  - Why this design?
    - Relativity: Using percentages rather than absolute values allows HPA to adapt to different Pod sizes
    - Flexibility: The same HPA configuration can apply to applications with different resource configurations
    - Predictability: Based on requests calculation, ensuring consistent behavior
- **Resource Reservation**: Even if the Pod doesn't use CPU, these resources are reserved and won't be allocated to other Pods

#### CPU Limits - 300m
```yaml
resources:
  limits:
    cpu: 300m
```

**Meaning:**
- **Resource Upper Limit**: Pod can use at most 300m CPU resources
- **Throttling Mechanism**: When Pod attempts to use more than 300m CPU, it will be throttled
- **Prevent Resource Abuse**: Prevents a single Pod from consuming too many CPU resources, affecting other Pods
- **Burst Usage**: Pod can use more than requests but less than limits for short periods

#### Practical Operation Example
- requests: 200m
- limits: 300m

| Load | CPU Usage | Calculation | HPA Behavior | System Status |
|------|-----------|-------------|--------------|---------------|
| 🔵 **Low** | 50m CPU | 50m ÷ 200m = 25% | Won't trigger HPA scale-down | Node still has 150m CPU reserved but unused |
| 🟢 **Normal** | 150m CPU | 150m ÷ 200m = 75% | If threshold is 70%, will trigger scale-up | Pod operates normally with sufficient resources |
| 🔴 **High** | Attempts 400m CPU | Actually limited to 300m (limits restriction) | Excess is throttled, may cause performance degradation | Pod performance limited, needs to adjust limits |

#### Configuration Recommendations

**Requests Configuration Principles:**
- Set to application's average CPU usage
- Too low: May cause Pod to not get sufficient resources
- Too high: Wastes node resources, reduces cluster utilization

**Limits Configuration Principles:**
- Usually set to 1.5-2 times requests
- Too low: May limit application performance
- Too high: Loses resource protection effect

**HPA Considerations:**
- HPA calculates percentage based on requests
- Setting `--cpu-percent=10` means triggering scale-up when CPU usage exceeds 10% of requests
- In this example, that's when CPU usage exceeds 20m (200m × 10%)

## Important Notes

1. **Metrics Server Must Function Properly**: HPA relies on Metrics Server to obtain CPU and memory usage metrics
2. **Resource Limit Configuration**: Must set `resources.requests` and `resources.limits` in Deployment
3. **CPU Percentage Calculation**: HPA calculates CPU usage percentage based on `requests` value
4. **Cooldown Time**: HPA has built-in cooldown time (about five minutes) to avoid frequent scaling operations
5. **Monitoring Interval**: Checks metrics every 15 seconds by default

## Troubleshooting

### Common Issues
1. **HPA Cannot Obtain Metrics**: Check if Metrics Server is functioning properly
2. **Scaling Not Working**: Confirm resource limits are set in Deployment
3. **Ingress Not Accessible**: Check Ingress controller status and network configuration