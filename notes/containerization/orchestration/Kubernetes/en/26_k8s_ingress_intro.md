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

Before introducing Ingress (each service has its own Load Balancer):
```
Internet
    |
    v
[Load Balancer A] -> [Service A]
[Load Balancer B] -> [Service B]
[Load Balancer C] -> [Service C]
```

After introducing Ingress:
```
External User → Load Balancer → Ingress Controller → Ingress → Service → Pod
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

### Ingress vs Ingress Controller

| Item | Ingress Controller (Security Guard) | Ingress (Access Card Rules) | 
|------|-------------------------------------|------------------------------|
| **Nature** | Program running in Kubernetes, usually a Pod/Deployment | A type of resource in Kubernetes | 
| **Function** | Component that actually executes Ingress rules | Defines routing rules and policies |
| **Analogy** | Like a "security guard" who checks access cards and enforces rules | Like "access card issuance rules" that specify who can enter which areas |

#### Relationship Between Both
- **Inseparable**: Ingress Controller and Ingress must both exist to function
- **Clear Division of Labor**: Ingress defines rules, Controller executes rules
- **Dynamic Collaboration**: Controller monitors Ingress changes and updates in real-time

### Working Process (Using NGINX Ingress Controller as Example)

#### 1. Register Itself as Ingress Controller on Startup
For example, you deploy a set of `nginx-ingress-controller` Deployment + Service using Helm or YAML.

This set of Pods automatically registers itself as an Ingress Controller with the K8S API Server.

#### 2. Continuously Monitor Changes to Ingress Resources
The Ingress Controller continuously watches all Ingress objects across all Namespaces in K8S (also monitors related Services/Secrets).

Whenever someone adds, modifies, or deletes an Ingress, the Controller receives notifications.

#### 3. Parse Ingress Rules
The Controller reads the content of Ingress resources, such as:
- Which hosts (domains) to handle
- Which paths should route to which Service/Port
- Whether to add SSL/TLS (where are the certificates)

#### 4. Automatically Generate Reverse Proxy Configuration Files
Using NGINX Controller as an example, it "translates" K8S Ingress content into its own NGINX configuration file (nginx.conf).

If rules change, the Controller dynamically reloads the configuration without needing to manually restart the Pod.

#### 5. Actually Receive External Requests and Proxy Them
External HTTP/HTTPS traffic passes through the load balancer or NodePort and is directed to the Ingress Controller's Service.

NGINX routes requests to the corresponding K8S Service according to the translated rules (then the Service sends them to Pods).

### Key Benefits of Using Ingress

- **Single Entry Point**: Consolidates external access through one interface
- **Cost Efficiency**: Reduces the need for multiple LoadBalancer services
- **Advanced Routing**: Supports host-based and path-based routing
- **SSL Termination**: Can handle HTTPS certificates centrally
- **Load Distribution**: Works with various load balancing algorithms