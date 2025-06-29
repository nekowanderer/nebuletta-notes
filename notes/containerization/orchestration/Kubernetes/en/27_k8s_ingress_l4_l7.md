# L4/L7 Network Concepts in Kubernetes Ingress

[English](../en/27_k8s_ingress_l4_l7.md) | [繁體中文](../zh-tw/27_k8s_ingress_l4_l7.md) | [日本語](../ja/27_k8s_ingress_l4_l7.md) | [Back to Index](../README.md)

### OSI Network Model Fundamentals

Before discussing L4/L7 concepts in Kubernetes Ingress, it's essential to understand the basic architecture of the OSI network model:

- **Layer 1 (Physical Layer)**: Cables, fiber optics, and other physical connections
- **Layer 2 (Data Link Layer)**: MAC addresses, Ethernet frames
- **Layer 3 (Network Layer)**: IP addresses, routing
- **Layer 4 (Transport Layer)**: TCP/UDP protocols, ports
- **Layer 5-6 (Session/Presentation Layer)**: Encryption, compression
- **Layer 7 (Application Layer)**: HTTP, HTTPS, FTP, and other application protocols

### Layer 4 (L4) Load Balancing

Layer 4 load balancing operates at the transport layer with the following characteristics:

**Operating Principles:**
- Makes routing decisions based on IP addresses and ports
- Does not inspect packet contents, only source and destination information
- Uses TCP/UDP connection information for load distribution

**Technical Features:**
- **High Performance**: Fast processing with low latency
- **Protocol Agnostic**: Can handle any TCP/UDP-based traffic
- **Simple Routing**: Only forwards based on network information

**Application in Kubernetes:**
```
External Traffic → LoadBalancer Service → Based on IP:Port → Backend Pod
```

**Use Cases:**
- Applications requiring high throughput
- Non-HTTP protocol services (such as databases, TCP services)
- Simple load distribution requirements

### Layer 7 (L7) Load Balancing

Layer 7 load balancing operates at the application layer with more intelligent routing capabilities:

**Operating Principles:**
- Inspects complete HTTP/HTTPS request content
- Routes based on URL paths, hostnames, request headers, etc.
- Can modify request and response content

**Technical Features:**
- **Intelligent Routing**: Makes complex routing decisions based on application layer information
- **Content Awareness**: Can process based on request content
- **Protocol Specific**: Primarily handles HTTP/HTTPS traffic

**Application in Kubernetes:**
```
HTTP Request → Ingress Controller → Based on Host/Path → Specific Service → Pod
```

**Routing Examples:**
- `api.example.com/users` → User Service
- `api.example.com/orders` → Order Service
- `admin.example.com/*` → Admin Service

### Configuration Methods in Kubernetes

**L4 Service (LoadBalancer Service):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-l4
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: my-app
```

**L7 Service (Ingress):**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-l7
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 80
```

### Practical Application Scenarios Comparison

| Consideration | L4 Load Balancing | L7 Load Balancing |
|---------------|-------------------|-------------------|
| **When to Choose** | • Need maximum performance and lowest latency<br>• Handling non-HTTP protocols<br>• Simple load distribution requirements<br>• No need for content-based routing | • Need routing based on URL paths or hostnames<br>• Implementing SSL termination<br>• Need application layer health checks<br>• Performing request modification or rewriting<br>• Implementing microservices architecture API Gateway |
| **Real-world Scenarios** | • Database connection pools (MySQL, PostgreSQL)<br>• TCP long connection services<br>• Game servers<br>• Real-time communication systems | • Web application routing<br>• RESTful API service distribution<br>• User-based routing (A/B testing)<br>• Centralized SSL certificate management |

### Performance and Functionality Trade-offs

| Feature | L4 Load Balancing | L7 Load Balancing |
|---------|-------------------|-------------------|
| Processing Speed | Fast | Slower |
| Feature Richness | Basic | Rich |
| Routing Intelligence | Simple | Complex |
| Protocol Support | All TCP/UDP | Mainly HTTP/HTTPS |
| Resource Consumption | Low | High |
| Configuration Complexity | Simple | Complex |

### Plain Language Analogy

| Role | Analogy | Items Checked | Corresponding Ingress Level |
|------|---------|---------------|----------------------------|
| Security Gate | Only recognizes "Can this access card open this door" - Checks floor and door number | Card number, door number → Like **IP address + port number** | **L4** |
| Reception Desk | Further confirms "Which department are you visiting, what's your appointment" - Checks visitor info and purpose | Visiting department, visitor name, appointment notes → Like **HTTP Host, Path, Header** | **L7** |

- L4 Ingress: Like a gate that only checks "Does the card number match this door" - Verifies TCP/UDP + port number and lets through.
- L7 Ingress: Like a reception desk that not only swipes the card but also checks which floor you're going to, department name, and even requires filling out visitor forms before forwarding.