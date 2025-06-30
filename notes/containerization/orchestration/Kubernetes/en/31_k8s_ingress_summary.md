# Kubernetes Ingress Summary

[English](../en/31_k8s_ingress_summary.md) | [繁體中文](../zh-tw/31_k8s_ingress_summary.md) | [日本語](../ja/31_k8s_ingress_summary.md) | [Back to Index](../README.md)

## Overview

This note summarizes the three main configuration methods for Kubernetes Ingress: Default Backend, Hostname-based, and Path-based. Each method has its specific use cases and configuration approaches.

## Comparison of Three Configuration Methods

### 1. Default Backend

#### Use Cases
- **404 Error Handling**: Provides default responses when requests don't match any rules
- **Maintenance Pages**: Displays unified maintenance pages during system maintenance
- **Error Handling**: Provides unified error pages or API responses
- **Simple Applications**: Only a single service needs to be exposed externally

#### YAML Configuration Example
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
      name: default-app-service
      port:
        number: 8080
```

#### Characteristics
- **Simplest Configuration**: Only requires specifying `defaultBackend`
- **Global Default**: All unmatched requests are directed to the default service
- **No Routing Rules**: Does not include the `rules` section
- **Single Entry Point**: Suitable for single applications or simple architectures

### 2. Hostname-based Routing

#### Use Cases
- **Multi-tenant Architecture**: Different customers use different domain names
- **Environment Separation**: beta.demo.com, prod.demo.com, etc.
- **Microservices Architecture**: Each service has an independent domain name
- **Brand Separation**: Different brands use different domain names

#### YAML Configuration Example
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
            name: beta-app-service
            port:
              number: 8080
  - host: prod.demo.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: prod-app-service
            port:
              number: 8080
```

#### Characteristics
- **Domain Separation**: Different domain names point to different services
- **Clear Separation**: Each environment or service has an independent domain name
- **Easy Management**: Can set different SSL certificates for each domain name
- **Suitable for Multi-environments**: Development, testing, production environment separation

### 3. Path-based Routing

#### Use Cases
- **Single Domain Multiple Services**: Providing multiple services under the same domain name
- **API Gateway**: Unified entry point, routing to different APIs by path
- **Frontend Routing**: Backend API routing for SPA applications
- **Version Control**: Different versions of APIs use different paths

#### YAML Configuration Example
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-path
  namespace: app-ns
spec:
  ingressClassName: nginx
  rules:
  - host: api.demo.com
    http:
      paths:
      - pathType: Prefix
        path: /beta
        backend:
          service:
            name: beta-app-service
            port:
              number: 8080
      - pathType: Prefix
        path: /prod
        backend:
          service:
            name: prod-app-service
            port:
              number: 8080
      - pathType: Prefix
        path: /api/v1
        backend:
          service:
            name: api-v1-service
            port:
              number: 8080
```

#### Characteristics
- **Path Separation**: Routing by path under the same domain name
- **Unified Entry**: All services share the same domain name
- **Flexible Configuration**: Can combine multiple path types
- **Suitable for API Design**: Standard practice for RESTful APIs

## Configuration Differences Comparison

### YAML Structure Differences

| Configuration Method | Main Section | Routing Rules | Backend Configuration |
|---------------------|--------------|---------------|----------------------|
| Default Backend | `defaultBackend` | None | Single service |
| Hostname-based | `rules[].host` | By domain name | One service per domain |
| Path-based | `rules[].http.paths` | By path | One service per path |

### Routing Priority

1. **Hostname-based**: Domain name matching takes priority
2. **Path-based**: Path matching under the same domain name
3. **Default Backend**: Used when all rules don't match

### Testing Method Differences

#### Default Backend
```bash
$ curl [ingress_ip]:80
```

#### Hostname-based
```bash
$ curl [ingress_ip]:80 -H 'Host: beta.demo.com'
$ curl [ingress_ip]:80 -H 'Host: prod.demo.com'
```

#### Path-based
```bash
$ curl [ingress_ip]:80/beta -H 'Host: api.demo.com'
$ curl [ingress_ip]:80/prod -H 'Host: api.demo.com'
```

## Selection Recommendations

### Choose Default Backend When:
- Only a single service needs to be exposed externally
- Need unified error handling mechanism
- System architecture is simple, no complex routing needed

### Choose Hostname-based When:
- Need to provide independent domain names for different environments or customers
- Need to set different SSL certificates for different domain names
- Want clear service separation

### Choose Path-based When:
- Want to use a single domain name to provide multiple services
- Designing RESTful API architecture
- Need version control or environment separation but want to share domain names

### Combined Usage
In practice, these three methods can be used in combination:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-combined
  namespace: app-ns
spec:
  ingressClassName: nginx
  defaultBackend:
    service:
      name: error-page-service
      port:
        number: 8080
  rules:
  - host: api.demo.com
    http:
      paths:
      - pathType: Prefix
        path: /v1
        backend:
          service:
            name: api-v1-service
            port:
              number: 8080
      - pathType: Prefix
        path: /v2
        backend:
          service:
            name: api-v2-service
            port:
              number: 8080
  - host: admin.demo.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: admin-service
            port:
              number: 8080
```

## Important Notes

1. **Path Priority**: In Path-based configuration, longer paths should be placed before shorter paths
2. **Host Header**: Testing must include the correct Host header
3. **SSL Certificates**: Hostname-based can set different SSL certificates for each domain name
4. **Namespace**: Ensure all related resources are in the correct namespace
5. **Service Availability**: Ensure backend services are running normally before deploying Ingress

## Summary

The three configuration methods of Kubernetes Ingress each have their applicable scenarios:

- **Default Backend**: Suitable for simple architectures and error handling
- **Hostname-based**: Suitable for multi-tenant and multi-environment separation
- **Path-based**: Suitable for API gateways and unified entry design

In practice, these methods can be combined according to requirements to build flexible and scalable traffic routing architectures.