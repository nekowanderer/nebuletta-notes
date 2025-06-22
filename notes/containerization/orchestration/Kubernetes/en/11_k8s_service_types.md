# K8S Service Types

[English](../en/11_k8s_service_types.md) | [繁體中文](../zh-tw/11_k8s_service_types.md) | [日本語](../ja/11_k8s_service_types.md) | [Back to Index](../README.md)

### ClusterIP
| Key Point | Description |
| --------- | ----------- |
| Role | "Internal Extension" between Pods in the cluster |
| Entry Point | Only accessible within the cluster (kube-proxy + iptables / IPVS DNAT) |
| DNS | Kubernetes' built-in CoreDNS automatically configures A/AAAA records, Pods call each other using `<svc>.<ns>.svc.cluster.local` |
| Suitable Scenarios | Microservices calling each other, internal APIs, databases, metrics, queues, and other services that are not intended to be directly exposed externally |

- Advantages
  - Available by default, minimal configuration required.
  - When Pods are rolling updated and IPs change, downstream code doesn't need to be modified; automatic forwarding through Service names.
- Disadvantages
  - Completely inaccessible from outside the cluster; testing or debugging requires additional port-forwarding or temporary Service creation.

### NodePort
| Key Point | Description |
| --------- | ----------- |
| Role | Uses each Worker Node as a temporary "gateway": opens fixed ports in the 30000-32767 range |
| Entry Point | External users → `Any node IP:NodePort` → Cluster Service → Pod |
| Suitable Scenarios | PoC, internal lab, no cloud Load Balancer, but need external (or VPN users) to access services quickly |

- Advantages
  - No cloud integration required, pure K8s functionality works.
  - One line of YAML can push services to every node for external access.
- Disadvantages
  - Port numbers are not user-friendly or memorable; corporate firewalls need to allow these high port ranges.
  - If there are many nodes, each listening on the same port, security audits become complex.
  - No advanced features like health checks, SSL, L7 forwarding, etc.

### LoadBalancer
| Key Point | Description |
| --------- | ----------- |
| Role | Calls cloud providers (AWS ELB, GCP TCP/HTTP LB, Azure LB...) to automatically generate an L4/7 Load Balancer |
| Entry Point | External users → LB public IP / DNS → forwarded to NodePort (managed by cloud) → Pod |
| Suitable Scenarios | Production environments, need fixed public DNS, TLS termination, health checks, high volume, cross-AZ high availability traffic |

- Advantages
  - One Service, one cloud LB, external access via DNS name.
  - Native support for health checks, SSL offload, Cross-Zone / AZ HA.
  - Can be combined with ExternalTrafficPolicy=Local for client IP preservation.
- Disadvantages
  - Each LB costs money; many microservices become expensive.
  - Limited to cloud environments that support LB; bare metal or on-prem requires MetalLB, BGP LB, or self-hosted solutions.
  - Creating/destroying LBs requires waiting for cloud APIs, deployment time longer than ClusterIP & NodePort.

### Comparison
| Feature Aspect | ClusterIP | NodePort | LoadBalancer |
| -------------- | --------- | -------- | ------------ |
| External Accessibility | No | Yes (IP:Port) | Yes (Public/Private LB DNS) |
| Connection Memorability | ★★★ | ★☆ (High port numbers) | ★★★ |
| Cost | Free | Free | Additional cost per LB |
| TLS / Health Checks | Handled within Pod | Handled within Pod | LB can handle |
| Cloud Dependency | None | None | High (requires cloud provider integration) |
| Common Use Cases | Internal microservices | Testing, Lab | Production external APIs, Web |

- LoadBalancer is the cloud's external facade
- NodePort is its unified backdoor to the cluster
- The actual forwarding work of Services relies on kube-proxy/iptables on nodes.

### Why does LoadBalancer forward to NodePort?
| Behavior | Where it happens | Why is this done? |
| --------- | ---------------- | ----------------- |
| Creating a Service with `type: LoadBalancer` | Kubernetes API Server ✅ | Just declares "I want a cloud LB" |
| Cloud Controller Manager (CCM) calls cloud provider API | Control Plane ✅ | Creates an ELB / NLB / L4 LB... |
| **Simultaneously** kube-proxy opens **NodePort** on all Worker Nodes (takes one number from 30000-32767) | **Each Worker Node** (Data Plane) | Provides a unified landing point for LB <br>→ LB only needs to know "node IP : NodePort" to send traffic into the cluster |

- Therefore: LoadBalancer ➜ NodePort ➜ ClusterIP ➜ Pod. NodePort is the "common language" between cloud LB and cluster.

### Are Services basically running inside worker nodes?
- Service is not a Pod or process; it has no container itself; it's a set of iptables / IPVS rules deployed by kube-proxy to each Worker Node.
- kube-proxy (or eBPF/XDP solutions) dynamically deploys iptables/IPVS/eBPF rules on each worker node, transforming:
  - Incoming `node-ip:nodeport` or container calls to `clusterip:port` into (DNAT) the actual Pod IP / port, combined with round-robin, weights, or session-affinity.
- In other words, whether you're hitting ClusterIP, NodePort, or external LB, packets are ultimately DNAT'd to Pod IP on nodes.
- Therefore, "Service runs on worker node" can be understood as: it relies on node-level network rules resident on each node, not as a server process on a specific host.

### Is NodePort by default where worker nodes open for external or LB connections?
| Characteristic | Description |
| -------------- | ----------- |
| Location | Each Worker Node will `LISTEN 0.0.0.0:<nodePort>` (handled by kube-proxy) |
| Default Range | `30000-32767` (can be changed via `--service-node-port-range`) |
| Exposure Source | - Directly declare `type: NodePort` for external or VPN users to access <br> - Implicitly created by `type: LoadBalancer`, only for cloud LB access |
| Traffic Flow | `Node IP:NodePort` → enters Service's DNAT → randomly selects a backend Pod (RR / IPVS LC...) |
| Common Misunderstanding | **NodePort ≠ opened on Pod**; it's opened on node network interface. Pods only receive packets after final DNAT |
| Always accessible externally? | **Still depends on firewall / Security Group / NACL** allowing; K8s only binds ports, doesn't help open security rules |

### Simplified Relationship Diagram
```
External client
     |
     |  (LoadBalancer DNS / IP)
[Cloud LB]  <----------------- Only appears with type=LoadBalancer
     |
nodeIP:NodePort  <------------ Shared by NodePort and LoadBalancer
     |
ClusterIP:Port   <------------ Internal cluster traffic (Service name resolution)
     |
PodIP:Port       <------------ Actual containers

```

### References
- [Kubernetes Official Document - Service Type](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types) 