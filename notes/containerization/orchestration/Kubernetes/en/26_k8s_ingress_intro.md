# Kubernetes Ingress Introduction

[English](../en/26_k8s_ingress_intro.md) | [繁體中文](../zh-tw/26_k8s_ingress_intro.md) | [日本語](../ja/26_k8s_ingress_intro.md) | [Back to Index](../README.md)

```
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                                Cluster                                  │
  │  ┌─────────────┐                                                        │
  │  │  Service A  │                                                        │
  │  │ ┌───┐ ┌───┐ │    all.demo.com/beta                                   │
  │  │ │Pod│ │Pod│ │ ←─────┐                                                │
  │  │ └───┘ └───┘ │       │                                                │
  │  └─────────────┘       │                                                │
  │                        │                                                │
  │                        │    ┌─────────┐   ┌────────────────────┐        │
  │                        └─── │         │   │                    │        │
  │                             │ Ingress │ ← │ Ingress Controller │        │
  │                        ┌─── │         │   │                    │        │
  │                        │    └─────────┘   └────────────────────┘        │
  │                        │                           ↑                    │
  │  ┌─────────────┐       │                           │                    │
  │  │  Service B  │       │                  ┌─────────────────┐           │
  │  │ ┌───┐ ┌───┐ │ ←─────┘                  │  Load Balancer  │ ←── User  │
  │  │ │Pod│ │Pod│ │    all.demo.com/prod     └─────────────────┘           │
  │  │ └───┘ └───┘ │                                                        │
  │  └─────────────┘                                                        │
  └─────────────────────────────────────────────────────────────────────────┘
```

### What is Kubernetes Ingress

Kubernetes Ingress is an API object that manages external access to services within a cluster. It acts as a single entry point for HTTP and HTTPS traffic, providing rules to route traffic to different services based on hostnames, paths, or other criteria.

### Basic Ingress Architecture

The basic Ingress setup consists of several key components:

- **Ingress Resource**: Defines the routing rules
- **Ingress Controller**: Implements the actual routing logic
- **Load Balancer**: External entry point for traffic
- **Services**: Backend services that receive routed traffic

```
External User → Load Balancer → Ingress Controller → Ingress → Services → Pod
```

### Single Service Routing with Default Backend

In the simplest configuration, Ingress can direct all traffic to a single service:

- External requests come through the Load Balancer
- Ingress Controller processes the requests
- All traffic is routed to a **default backend** service
- The service then forwards traffic to its associated Pods (Service A with multiple Pods)

This setup is useful for simple applications that don't require complex routing rules.

### Multiple Services with Host-based Routing

Ingress enables routing to different services based on different hostnames:

- **all.demo.com/beta** routes to Service A
- **all.demo.com/prod** routes to Service B

Each service maintains its own set of Pods, allowing for:
- Environment separation (beta vs production)
- Independent scaling of services
- Path-based routing within the same domain

### Ingress Controller Options

**Cloud Provider Solutions:**
- **AWS ALB** (Application Load Balancer)
- **GCP Load Balancer** 
- **Azure Application Gateway**

**Self-managed Solutions:**
- **Local Nginx** - Most common open-source option

The choice of Ingress Controller depends on:
- Infrastructure platform (cloud vs on-premises)
- Feature requirements (SSL termination, authentication, etc.)
- Performance and cost considerations

### Key Benefits of Using Ingress

- **Single Entry Point**: Consolidates external access through one interface
- **Cost Efficiency**: Reduces the need for multiple LoadBalancer services
- **Advanced Routing**: Supports host-based and path-based routing
- **SSL Termination**: Can handle HTTPS certificates centrally
- **Load Distribution**: Works with various load balancing algorithms