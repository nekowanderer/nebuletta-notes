# Kubernetes Ingress Path-based Implementation Guide

[English](../en/30_k8s_ingress_path_based.md) | [繁體中文](../zh-tw/30_k8s_ingress_path_based.md) | [日本語](../ja/30_k8s_ingress_path_based.md) | [Back to Index](../README.md)

## Overview

Path-based Ingress is an important traffic routing method in Kubernetes that allows requests to be forwarded to different backend services based on URL paths. This guide demonstrates how to implement Path-based Ingress in a Minikube environment, routing `/beta` and `/prod` paths to different application services respectively.

## Environment Setup

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

### Enable Ingress Addon

```bash
$ minikube addons enable ingress
$ kubectl get all -n ingress-nginx
```

## Application Deployment

### Deploy Beta Application

```bash
$ kubectl apply -f beta-app-all.yaml
$ kubectl get all -n app-ns
```

### Deploy Production Application

```bash
$ kubectl apply -f prod-app-all.yaml
$ kubectl get all -n app-ns
```

## Ingress Configuration

### Path-based Ingress Configuration File

Create the `ingress-path.yaml` file:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-path
  namespace: app-ns
spec:
  ingressClassName: nginx
  rules:
  - host: all.demo.com
    http:
      paths:
      - pathType: Prefix
        path: /beta
        backend:
          service:
            name: beta-app-service-clusterip
            port:
              number: 8080
      - pathType: Prefix
        path: /prod
        backend:
          service:
            name: prod-app-service-clusterip
            port:
              number: 8080
```

### Deploy Ingress

```bash
$ kubectl apply -f ingress-path.yaml

$ kubectl get ingress -n app-ns
NAME               CLASS   HOSTS                         ADDRESS        PORTS   AGE
ingress-hostname   nginx   beta.demo.com,prod.demo.com   192.168.49.2   80      26m
ingress-path       nginx   all.demo.com,prod.demo.com    192.168.49.2   80      63s
```

## Configuration Explanation

### Path-based Routing Rules

- **`/beta` path**: Forwards to `beta-app-service-clusterip` service
- **`/prod` path**: Forwards to `prod-app-service-clusterip` service
- **`pathType: Prefix`**: Uses prefix matching, `/beta` will match `/beta`, `/beta/`, `/beta/api`, etc.

### Key Components

- **`ingressClassName: nginx`**: Specifies the use of NGINX Ingress Controller
- **`host: all.demo.com`**: Sets the virtual hostname
- **`pathType: Prefix`**: Path matching type is prefix matching

## Testing and Verification

### 1. Get Ingress IP

```bash
$ kubectl get ingress -n app-ns
```

### 2. Test Beta Application

```bash
$ curl [ingress_ip]:80/beta -H 'Host: all.demo.com'
```

### 3. Test Production Application

```bash
$ curl [ingress_ip]:80/prod -H 'Host: all.demo.com'
```

## Resource Cleanup

After completing tests, clean up all resources:

```bash
$ kubectl delete ns app-ns
```

## Important Notes

1. **Path Priority**: The Ingress controller matches paths in configuration order; longer paths should be placed before shorter ones
2. **Host Header**: Testing must include the correct Host header, otherwise requests may not route correctly
3. **Namespace**: Ensure all resources are in the same namespace, or specify namespaces correctly
4. **Service Availability**: Ensure backend services are running normally before deploying Ingress

## Frequently Asked Questions

Q: Why is it necessary to set the Host header?
A: The Ingress rule sets `host: all.demo.com`, so testing must include this Host header to correctly match the rule.

Q: How to add more paths?
A: Add new path configurations to the `paths` array, specifying `path`, `pathType`, and corresponding backend service.

Q: How to set different path types?
A: `pathType` can be set to:
- `Prefix`: Prefix matching
- `Exact`: Exact matching
- `ImplementationSpecific`: Implementation-specific matching