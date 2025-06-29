# Kubernetes Ingress Default Backend Implementation Guide

[English](../en/28_k8s_ingress_default_backend.md) | [繁體中文](../zh-tw/28_k8s_ingress_default_backend.md) | [日本語](../ja/28_k8s_ingress_default_backend.md) | [Back to Index](../README.md)

## Overview
This note explains how to configure Ingress defaultBackend in Kubernetes. When request paths don't match any rules, traffic will be directed to the default backend service.

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

### 3. Enable Ingress Addon
```bash
$ minikube addons enable ingress
💡  ingress is an addon maintained by Kubernetes. For any concerns contact minikube on GitHub.
You can view the list of minikube maintainers at: https://github.com/kubernetes/minikube/blob/master/OWNERS
    ▪ Using image registry.k8s.io/ingress-nginx/kube-webhook-certgen:v1.5.3
    ▪ Using image registry.k8s.io/ingress-nginx/controller:v1.12.2
    ▪ Using image registry.k8s.io/ingress-nginx/kube-webhook-certgen:v1.5.3
🔎  Verifying ingress addon...
🌟  The 'ingress' addon is enabled

$ kubectl get all -n ingress-nginx
NAME                                           READY   STATUS      RESTARTS   AGE
pod/ingress-nginx-admission-create-5fj8c       0/1     Completed   0          64s
pod/ingress-nginx-admission-patch-529hj        0/1     Completed   1          64s
pod/ingress-nginx-controller-67c5cb88f-dv4px   1/1     Running     0          64s

NAME                                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx-controller             NodePort    10.102.245.181   <none>        80:30447/TCP,443:31545/TCP   64s
service/ingress-nginx-controller-admission   ClusterIP   10.96.249.142    <none>        443/TCP                      64s

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-nginx-controller   1/1     1            1           64s

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-nginx-controller-67c5cb88f   1         1         1       64s

NAME                                       STATUS     COMPLETIONS   DURATION   AGE
job.batch/ingress-nginx-admission-create   Complete   1/1           7s         64s
job.batch/ingress-nginx-admission-patch    Complete   1/1           7s         64s
```

## Deploy Applications

### 1. Deploy Test Application
```bash
$ kubectl apply -f beta-app-all.yaml
namespace/app-ns created
deployment.apps/beta-app-deployment created
service/beta-app-service-clusterip created

$ kubectl get all -n app-ns
NAME                                       READY   STATUS    RESTARTS   AGE
pod/beta-app-deployment-865c646d9d-6f8sj   1/1     Running   0          31s
pod/beta-app-deployment-865c646d9d-c9ltr   1/1     Running   0          31s
pod/beta-app-deployment-865c646d9d-zfk9f   1/1     Running   0          31s

NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/beta-app-service-clusterip   ClusterIP   10.108.189.68   <none>        8080/TCP   31s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/beta-app-deployment   3/3     3            3           31s

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/beta-app-deployment-865c646d9d   3         3         3       31s
```

## Configure Ingress Default Backend

### 1. Create Ingress Configuration File
Create `ingress-defaultbackend.yaml` file:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-defaultbackend
  namespace: app-ns
spec:
  ingressClassName: nginx
  defaultBackend:
    service:
      name: beta-app-service-clusterip
      port:
        number: 8080
```

### 2. Deploy Ingress
```bash
$ kubectl apply -f ingress-defaultbackend.yaml

$ kubectl get ingress -n app-ns

$ kubectl describe ingress ingress-defaultbackend -n app-ns

$ kubectl get ingress -n app-ns -w
```

## Testing and Verification

### 1. Get Ingress IP
```bash
$ kubectl get ingress -n app-ns
```

### 2. Test Connection
```bash
$ curl [ingress_ip]:80
$ curl [ingress_ip]:80
$ curl [ingress_ip]:80
$ curl [ingress_ip]:80
$ curl [ingress_ip]:80
```

## Important Notes

- **defaultBackend Function**: When request paths don't match any Ingress rules, traffic will be directed to the specified default backend service
- **Use Cases**: Suitable for applications that need to handle 404 errors or provide default responses
- **Considerations**: Ensure the specified backend service exists and is running properly

## Cleanup Resources
```bash
$ kubectl delete -f ingress-defaultbackend.yaml

$ kubectl delete -f beta-app-all.yaml

$ minikube stop
```