# Kubernetes Service LoadBalancer Implementation Guide

[English](../en/15_k8s_loadbalancer_mode.md) | [繁體中文](../zh-tw/15_k8s_loadbalancer_mode.md) | [日本語](../ja/15_k8s_loadbalancer_mode.md) | [Back to Index](../README.md)

## Overview
LoadBalancer type Service is one of the most commonly used external access methods in Kubernetes. It automatically creates an external load balancer to distribute traffic to backend Pods.

## Prerequisites

##### 1. Start Docker Service
```bash
$ sudo service docker start
$ docker ps
```

##### 2. Create Minikube Cluster
```bash
$ minikube start --driver docker
$ minikube status
```

## Deploy Application

Reuse the `simple-deployment.yaml` from [10_deploy_deployments](./10_deploy_deployments.md)

##### 1. Deploy Deployment
```bash
$ kubectl apply -f simple-deployment.yaml
$ kubectl get deployments
```

##### 2. Create LoadBalancer Service

Create `simple-service-loadbalancer.yaml` file:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: app-pod
  ports:
    - protocol: TCP
      port: 8080        # Port exposed by the Service
      targetPort: 80    # Port inside the Pod
```

##### 3. Deploy Service
```bash
$ kubectl apply -f simple-service-loadbalancer.yaml

$ kubectl get services
NAME                       TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
app-service-loadbalancer   LoadBalancer   10.110.110.138   <pending>     8080:31830/TCP   6s  # Note: EXTERNAL-IP is pending
app-service-nodeport       NodePort       10.106.158.207   <none>        8080:30080/TCP   26m
kubernetes                 ClusterIP      10.96.0.1        <none>        443/TCP          26h

$ kubectl describe service app-service-loadbalancer
Name:                     app-service-loadbalancer
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=app-pod
Type:                     LoadBalancer
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.110.110.138
IPs:                      10.110.110.138
Port:                     <unset>  8080/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  31830/TCP
Endpoints:                10.244.0.22:80,10.244.0.21:80,10.244.0.20:80
Session Affinity:         None
External Traffic Policy:  Cluster
Internal Traffic Policy:  Cluster
Events:                   <none>
```

## Enable External Access

##### 1. Start Minikube Tunnel
Execute in another terminal window:
```bash
$ minikube tunnel
Status:
        machine: minikube
        pid: 25727
        route: 10.96.0.0/12 -> 192.168.49.2
        minikube: Running
        services: [app-service-loadbalancer]
    errors:
                minikube: no errors
                router: no errors
                loadbalancer emulator: no errors
```

This command creates a network tunnel that simulates a load balancer locally, directing traffic to the minikube cluster.

##### 2. Test Load Balancing Function
```bash
# Check service status
$ kubectl get services
NAME                       TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)          AGE
app-service-loadbalancer   LoadBalancer   10.110.110.138   10.110.110.138   8080:31830/TCP   2m23s # EXTERNAL-IP is no longer pending, now bound
app-service-nodeport       NodePort       10.106.158.207   <none>           8080:30080/TCP   28m
kubernetes                 ClusterIP      10.96.0.1        <none>           443/TCP          26h

# Test connection (repeat to observe load balancing effect)
$ curl [service_external_ip]:8080
$ curl [service_external_ip]:8080
$ curl [service_external_ip]:8080
$ curl [service_external_ip]:8080
$ curl [service_external_ip]:8080
```

## Resource Cleanup

##### 1. Delete All Resources
```bash
$ kubectl delete all --all
pod "app-deployment-8687d78656-d6x8t" deleted
pod "app-deployment-8687d78656-gp96l" deleted
pod "app-deployment-8687d78656-p7k6j" deleted
service "app-service-loadbalancer" deleted
service "app-service-nodeport" deleted
service "kubernetes" deleted
deployment.apps "app-deployment" deleted
```

##### 2. Stop Minikube Tunnel
Press `Ctrl + C` in the terminal running `minikube tunnel`

## Key Concepts

- **LoadBalancer Type**: Automatically creates an external load balancer
- **port**: Port exposed by the Service
- **targetPort**: Port actually listening inside the Pod
- **Minikube Tunnel**: Simulates cloud load balancer functionality in local environment

## Important Notes

1. LoadBalancer type requires cloud provider support
2. In Minikube environment, `minikube tunnel` is needed to simulate external access
3. In actual production environments, LoadBalancer will automatically assign external IP addresses 