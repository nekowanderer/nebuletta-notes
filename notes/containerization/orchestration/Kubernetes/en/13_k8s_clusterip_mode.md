# Kubernetes Service ClusterIP Implementation Guide

[English](../en/13_k8s_clusterip_mode.md) | [繁體中文](../zh-tw/13_k8s_clusterip_mode.md) | [日本語](../ja/13_k8s_clusterip_mode.md) | [Back to Index](../README.md)

## Overview

ClusterIP is the most basic Service type in Kubernetes, primarily used for communication between Pods within the cluster. This guide demonstrates how to deploy and test a ClusterIP Service in practice.

## Prerequisites

### 1. Start Docker Environment
```bash
# Start Docker service
$ sudo service docker start

# Verify Docker is running
$ docker ps
```

### 2. Create Minikube Cluster
```bash
# Start Minikube with Docker driver
$ minikube start --driver docker

# Check cluster status
$ minikube status
```

## Deploy Application

Reuse the `simple-deployment.yaml` from [10_deploy_deployments](./10_deploy_deployments.md)

### 1. Deploy Deployment
```bash
# Deploy application
$ kubectl apply -f simple-deployment.yaml

# Check Deployment status
$ kubectl get deployments
```

## Create ClusterIP Service

### 1. Create Service Configuration
Create `simple-service-clusterip.yaml` file:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service-clusterip
spec:
  type: ClusterIP
  selector:
    app: app-pod
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
```

### 2. Deploy Service
```bash
# Deploy ClusterIP Service
$ kubectl apply -f simple-service-clusterip.yaml

# View all Services
$ kubectl get services
NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
app-service-clusterip   ClusterIP   10.105.138.111   <none>        8080/TCP   7s
kubernetes              ClusterIP   10.96.0.1        <none>        443/TCP    25h

# View Service details
$ kubectl describe service app-service-clusterip
Name:                     app-service-clusterip
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=app-pod
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.105.138.111
IPs:                      10.105.138.111
Port:                     <unset>  8080/TCP
TargetPort:               80/TCP
Endpoints:                10.244.0.22:80,10.244.0.21:80,10.244.0.20:80
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>
```

## Test ClusterIP Connectivity

### 1. Get Service IP
```bash
# View Service information to get ClusterIP
$ kubectl get services
```

### 2. Test from Minikube Container
```bash
# View Docker containers
$ docker ps

# Enter Minikube container
$ docker exec -it minikube bash

# Test connectivity using Service IP (replace [service_ip] with actual ClusterIP)
$ curl [service_ip]:8080

# Exit container
$ exit
```

## Key Concepts

### ClusterIP Characteristics
- **Internal Cluster Communication**: Only accessible within the Kubernetes cluster
- **Automatic Load Balancing**: kube-proxy automatically distributes traffic to backend Pods
- **DNS Resolution**: Service names can be resolved via DNS within the cluster
- **Fixed IP**: Service IP remains constant within the cluster, even when Pods restart

### Network Flow
```
Pod A → ClusterIP Service → Pod B
```

### Use Cases
- Internal communication between microservices
- Database connections
- Internal API calls
- Monitoring and logging services

## Troubleshooting

### Common Issues
1. **Service Cannot Connect**
   - Check if Pod labels match Service selector
   - Verify Pods are running and healthy

2. **Port Mismatch**
   - Ensure Service `targetPort` matches Pod container port

3. **DNS Resolution Issues**
   - Use full DNS name within cluster: `service-name.namespace.svc.cluster.local`

## Related Resources

- [Kubernetes Service Official Documentation](https://kubernetes.io/docs/concepts/services-networking/service/)
- [ClusterIP Service Type Explanation](../11_k8s_service_types.md) 