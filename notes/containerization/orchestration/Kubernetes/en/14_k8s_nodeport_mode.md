# Kubernetes Service NodePort Implementation Guide

[English](../en/14_k8s_nodeport_mode.md) | [繁體中文](../zh-tw/14_k8s_nodeport_mode.md) | [日本語](../ja/14_k8s_nodeport_mode.md) | [Back to Index](../README.md)

## Overview

NodePort is a type of Kubernetes Service that opens a specific port (range 30000-32767) on each node, allowing external traffic to access services within the cluster through the node IP and that port.

## Prerequisites

##### 1. Start Docker Environment
```bash
# Start Docker service
$ sudo service docker start

# Verify Docker is running
$ docker ps
```

##### 2. Create Minikube Cluster
```bash
# Start Minikube with Docker driver
$ minikube start --driver docker

# Check cluster status
$ minikube status
```

## Deploy Application

Reuse the `simple-deployment.yaml` from [10_deploy_deployments](./10_deploy_deployments.md)

##### 1. Deploy Deployment
```bash
# Deploy application
$ kubectl apply -f simple-deployment.yaml

# Check deployment status
$ kubectl get deployments
```

## Create NodePort Service

##### 1. Create Service Configuration File
Create `simple-service-nodeport.yaml` file:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service-nodeport
spec:
  type: NodePort
  selector:
    app: app-pod
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
      nodePort: 30080  # Optional: specify a particular NodePort, range 30000-32767
```

##### 2. Deploy Service
```bash
# Deploy NodePort Service
$ kubectl apply -f simple-service-nodeport.yaml

# View all services
$ kubectl get services
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
app-service-nodeport   NodePort    10.106.158.207   <none>        8080:30080/TCP   5s  # <- 30080 is the node port
kubernetes             ClusterIP   10.96.0.1        <none>        443/TCP          25h

# View service details
$ kubectl describe service app-service-nodeport
Name:                     app-service-nodeport
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=app-pod
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.106.158.207
IPs:                      10.106.158.207
Port:                     <unset>  8080/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  30080/TCP
Endpoints:                10.244.0.20:80,10.244.0.22:80,10.244.0.21:80
Session Affinity:         None
External Traffic Policy:  Cluster
Internal Traffic Policy:  Cluster
Events:                   <none>
```

## Get Node Information

##### 1. View Node Information
```bash
# View all nodes
$ kubectl get nodes
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   25h   v1.33.1

# View node details to get node IP
$ kubectl describe node minikube
...
ddresses:
  InternalIP:  192.168.49.2 # <- copy this, this is the node IP
  Hostname:    minikube
...
```

##### 2. Get Node IP
From the output of `kubectl describe node minikube`, find the `InternalIP` or `ExternalIP` field (in this example, it's `InternalIP`).

## Test NodePort Connection

##### 1. Test Using Node IP and NodePort
```bash
# Test using node IP and NodePort
# Format: curl [node_ip]:[nodeport]
$ curl [node_ip]:30080
```

##### 2. Use Minikube Service URL
```bash
# Minikube provides convenient service URL functionality
$ minikube service app-service-nodeport --url

# Test using the returned URL
$ curl $(minikube service app-service-nodeport --url)
```

## Important Concepts

##### NodePort Characteristics
- **External Access**: Can access services from outside the cluster through node IP and NodePort
- **Port Range**: NodePort uses ports in the range 30000-32767
- **Node Level**: Each node opens the same NodePort
- **Load Balancing**: kube-proxy automatically distributes traffic to backend pods

##### Network Flow
```
External Client → Node IP:NodePort → kube-proxy → Pod
```

##### Use Cases
- Development environment testing
- Internal network access
- Scenarios that don't require cloud load balancers
- Specific node access requirements

## Advanced Configuration

##### 1. Specify NodePort
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service-nodeport
spec:
  type: NodePort
  selector:
    app: app-pod
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
      nodePort: 30080  # Specify a particular NodePort
```

##### 2. External Traffic Policy
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service-nodeport
spec:
  type: NodePort
  externalTrafficPolicy: Local  # or Cluster
  selector:
    app: app-pod
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
```

## Troubleshooting

##### Common Issues
1. **Cannot connect from external**
   - Verify firewall settings allow traffic in NodePort range
   - Check if node IP is correct
   - Confirm pods are running and healthy

2. **NodePort conflicts**
   - Check if other services are using the same NodePort
   - Can manually specify different NodePort

3. **Port range issues**
   - NodePort must be within 30000-32767 range
   - Avoid using system reserved ports

##### Debug Commands
```bash
# Check service status
$ kubectl get services

# Check endpoints
$ kubectl get endpoints app-service-nodeport

# Check pod status
$ kubectl get pods -l app=app-pod

# View service details
$ kubectl describe service app-service-nodeport
```

## Related Resources

- [Kubernetes Service Official Documentation](https://kubernetes.io/docs/concepts/services-networking/service/)
- [NodePort Service Type Explanation](./11_k8s_service_types.md)
- [DNAT and SNAT Operations in Kubernetes](./12_dnat_and_snat_in_k8s.md) 