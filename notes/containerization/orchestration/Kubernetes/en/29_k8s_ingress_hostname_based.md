# Kubernetes Ingress Hostname-based Implementation Guide

[English](../en/29_k8s_ingress_hostname_based.md) | [繁體中文](../zh-tw/29_k8s_ingress_hostname_based.md) | [日本語](../ja/29_k8s_ingress_hostname_based.md) | [Back to Index](../README.md)

## Overview

This note explains how to use Ingress controllers in Kubernetes to implement hostname-based routing, allowing different domain names to point to different services.

## Prerequisites

### 1. Start Docker and Minikube

```bash
# Start Docker service
$ sudo service docker start
$ docker ps

# Create Minikube cluster
$ minikube start --driver docker
$ minikube status
```

### 2. Enable Ingress Addon

```bash
# Enable minikube ingress addon
$ minikube addons enable ingress
$ kubectl get all -n ingress-nginx
NAME                                           READY   STATUS      RESTARTS      AGE
pod/ingress-nginx-admission-create-5fj8c       0/1     Completed   0             22h
pod/ingress-nginx-admission-patch-529hj        0/1     Completed   1             22h
pod/ingress-nginx-controller-67c5cb88f-dv4px   1/1     Running     1 (14m ago)   22h

NAME                                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx-controller             NodePort    10.102.245.181   <none>        80:30447/TCP,443:31545/TCP   22h
service/ingress-nginx-controller-admission   ClusterIP   10.96.249.142    <none>        443/TCP                      22h

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-nginx-controller   1/1     1            1           22h

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-nginx-controller-67c5cb88f   1         1         1       22h

NAME                                       STATUS     COMPLETIONS   DURATION   AGE
job.batch/ingress-nginx-admission-create   Complete   1/1           7s         22h
job.batch/ingress-nginx-admission-patch    Complete   1/1           7s         22h
```

## Deploy Applications

### 1. Deploy Beta Environment Application

```bash
# Deploy all beta environment resources
$ kubectl apply -f beta-app-all.yaml
$ kubectl get all -n app-ns
NAME                                       READY   STATUS    RESTARTS   AGE
pod/beta-app-deployment-865c646d9d-7lvf2   1/1     Running   0          2m35s
pod/beta-app-deployment-865c646d9d-ffpzt   1/1     Running   0          2m35s
pod/beta-app-deployment-865c646d9d-npjt2   1/1     Running   0          2m35s
pod/prod-app-deployment-586c5dcc59-dctfp   1/1     Running   0          37s
pod/prod-app-deployment-586c5dcc59-dhpr7   1/1     Running   0          37s
pod/prod-app-deployment-586c5dcc59-v9hm7   1/1     Running   0          37s

NAME                                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/beta-app-service-clusterip   ClusterIP   10.105.100.5     <none>        8080/TCP   2m35s
service/prod-app-service-clusterip   ClusterIP   10.108.143.220   <none>        8080/TCP   37s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/beta-app-deployment   3/3     3            3           2m35s
deployment.apps/prod-app-deployment   3/3     3            3           37s

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/beta-app-deployment-865c646d9d   3         3         3       2m35s
replicaset.apps/prod-app-deployment-586c5dcc59   3         3         3       37s
```

### 2. Deploy Production Environment Application

First copy and modify the configuration file:

```bash
# Copy beta configuration to prod configuration
$ cp beta-app-all.yaml prod-app-all.yaml
$ vi prod-app-all.yaml
```

**prod-app-all.yaml content:**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app-ns
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prod-app-deployment
  namespace: app-ns
spec:
  replicas: 3
  selector:
    matchLabels:
      app: prod-app-pod
  template:
    metadata:
      labels:
        app: prod-app-pod
    spec:
      containers:
      - name: prod-app-container
        image: uopsdod/k8s-hostname-amd64-prod:v1
        ports: 
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: prod-app-service-clusterip
  namespace: app-ns
spec:
  selector:
    app: prod-app-pod
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
```

Deploy production environment:

```bash
# Deploy all prod environment resources
$ kubectl apply -f prod-app-all.yaml
$ kubectl get all -n app-ns
```

## Configure Ingress Routing

### Create Ingress Configuration File

**ingress-hostname.yaml content:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-hostname
  namespace: app-ns
spec:
  ingressClassName: nginx
  rules:
  - host: beta.demo.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: beta-app-service-clusterip
            port:
              number: 8080
  - host: prod.demo.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: prod-app-service-clusterip
            port:
              number: 8080
```

### Deploy Ingress

```bash
# Deploy Ingress configuration
$ kubectl apply -f ingress-hostname.yaml
ingress.networking.k8s.io/ingress-hostname created

$ kubectl get all -n app-ns

$ kubectl get ingress -n app-ns
NAME               CLASS   HOSTS                         ADDRESS        PORTS   AGE
ingress-hostname   nginx   beta.demo.com,prod.demo.com   192.168.49.2   80      93s
```

## Testing and Verification

### Test Connections

Use curl commands to test routing for different hostnames:

```bash
# Test beta environment
$ curl [ingress_ip]:80 -H 'Host: beta.demo.com'
$ curl [ingress_ip]:80 -H 'Host: beta.demo.com'
$ curl [ingress_ip]:80 -H 'Host: beta.demo.com'

# Test production environment
$ curl [ingress_ip]:80 -H 'Host: prod.demo.com'
$ curl [ingress_ip]:80 -H 'Host: prod.demo.com'
$ curl [ingress_ip]:80 -H 'Host: prod.demo.com'
```

## Architecture Explanation

### Routing Mechanism

- **beta.demo.com** → Routes to `beta-app-service-clusterip`
- **prod.demo.com** → Routes to `prod-app-service-clusterip`

### Key Components

1. **Ingress Controller**: Uses nginx as the Ingress controller
2. **Host Rules**: Routes based on HTTP Host header
3. **Service Backend**: Each hostname corresponds to a different ClusterIP Service