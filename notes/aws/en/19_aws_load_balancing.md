# AWS Load Balancing Guide

[English](../en/19_aws_load_balancing.md) | [繁體中文](../zh-tw/19_aws_load_balancing.md) | [日本語](../ja/19_aws_load_balancing.md) | [Back to Index](../README.md)

### AWS Load Balancer Types Comparison

AWS provides three main types of load balancers, each with specific use cases and functional characteristics:

| Type | Use Case | Supported Protocols | Operating Layer | Key Features |
|------|----------|-------------------|----------------|--------------|
| **Application Load Balancer (ALB)** | HTTP/HTTPS applications | HTTP, HTTPS, WebSocket | Layer 7 (Application) | Path routing, host routing, Lambda integration |
| **Network Load Balancer (NLB)** | High-performance TCP/UDP traffic | TCP, UDP, TLS | Layer 4 (Transport) | Ultra-low latency, static IP, extreme throughput |
| **Gateway Load Balancer (GLB)** | Third-party network appliances | GENEVE | Layer 3 (Network) | Firewall, IDS/IPS, DPI appliance integration |

### Load Balancer Core Functions

#### Traffic Distribution Algorithms
- **Round Robin**: Distribute requests sequentially to each target
- **Least Outstanding Requests**: Route to target with fewest active requests
- **Flow Hash**: Distribute based on hash of source IP, destination IP, and protocol

#### Health Check Mechanism
- **Health Check Path**: Periodically check if targets respond normally
- **Healthy Threshold**: Number of consecutive successes required to mark as healthy
- **Unhealthy Threshold**: Number of consecutive failures required to mark as unhealthy
- **Check Interval**: Frequency of health check settings

---

## Target Groups

### Target Group Association with ALB

#### Basic Relationship
- **ALB cannot route directly to EC2 or services**
- **Must use Target Group as intermediary**
- Target Group serves as ALB's "target container"

#### Association Architecture
```
Internet → ALB → Listener → Rules → Target Group → Targets (EC2/IP/Lambda)
```

#### ALB Listener and Target Group Integration
```bash
# ALB Listener forwards traffic to Target Group
$ aws elbv2 create-listener \
    --load-balancer-arn arn:aws:elasticloadbalancing:region:account:loadbalancer/app/my-alb/1234567890123456 \
    --protocol HTTP \
    --port 80 \
    --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:region:account:targetgroup/my-targets/1234567890123456
```

#### Multiple Target Groups with Single ALB
- **Path Routing**: `/api/*` → API Target Group
- **Host Routing**: `admin.example.com` → Admin Target Group
- **Default Routing**: All other requests → Default Target Group

### Practical Application Examples

#### Microservices Architecture
```
ALB
├── /api/users/* → User Service Target Group
├── /api/orders/* → Order Service Target Group  
├── /admin/* → Admin Panel Target Group
└── /* → Frontend Target Group
```

#### Blue-Green Deployment
```
ALB
├── 90% traffic → Production Target Group (Blue)
└── 10% traffic → Staging Target Group (Green)
```

### Load Balancing Process

1. **ALB receives request**
2. **Listener checks port and protocol**
3. **Rules determine which Target Group to forward to**
4. **Target Group selects healthy target based on algorithm**
5. **Forward request to selected target**

### Target Group Role in ALB

| Function | Target Group Role |
|----------|-------------------|
| **Health Check** | Continuously monitor backend service status |
| **Load Distribution** | Distribute traffic based on algorithm |
| **Service Discovery** | Dynamically register/deregister targets |
| **Session Stickiness** | Maintain user session consistency |

### Target Group Types

#### Instance Target Group
- **Target Type**: EC2 instances
- **Registration Method**: Directly specify EC2 instance ID
- **Use Case**: Traditional EC2 architecture, needs Auto Scaling Group integration

#### IP Target Group
- **Target Type**: IP addresses
- **Registration Method**: Directly specify IP address and port
- **Use Case**: Containerized applications, microservices architecture, cross-VPC communication

#### Lambda Target Group
- **Target Type**: Lambda functions
- **Registration Method**: Specify Lambda function ARN
- **Use Case**: Serverless architecture, event-driven HTTP APIs

### Target Group Health Check Configuration

```bash
# Health check configuration example
$ aws elbv2 modify-target-group \
    --target-group-arn arn:aws:elasticloadbalancing:region:account:targetgroup/my-targets/1234567890123456 \
    --health-check-path /health \
    --health-check-interval-seconds 30 \
    --health-check-timeout-seconds 5 \
    --healthy-threshold-count 2 \
    --unhealthy-threshold-count 5
```

### Target Group Attribute Configuration

| Attribute | Description | Default Value | Recommended Setting |
|-----------|-------------|---------------|-------------------|
| **deregistration_delay.timeout_seconds** | Deregistration wait time | 300 seconds | Adjust based on application shutdown time |
| **stickiness.enabled** | Enable session stickiness | false | Set to true for stateful applications |
| **stickiness.type** | Stickiness type | lb_cookie | Application cookie or load balancer cookie |
| **load_balancing.algorithm.type** | Load balancing algorithm | round_robin | Choose based on application characteristics |

---

## Trust Stores

### Trust Store Concepts

Trust Store is an advanced feature of AWS Application Load Balancer used for managing and validating client certificates:

#### Main Functions
- **Client Certificate Validation**: Validate client certificates connecting to ALB
- **mTLS Support**: Implement mutual TLS authentication
- **Certificate Chain Validation**: Validate complete certificate chain
- **Certificate Revocation Check**: Support CRL and OCSP validation

#### Use Cases
- **Enterprise Internal APIs**: Require mandatory client certificate authentication
- **B2B Integration**: Secure API communication with partners
- **Compliance Requirements**: Meet specific industry security standards
- **Zero Trust Architecture**: Implement end-to-end certificate validation

### Trust Store Configuration Process

#### 1. Create Trust Store
```bash
$ aws elbv2 create-trust-store \
    --name my-trust-store \
    --ca-certificates-bundle-s3-bucket my-ca-bucket \
    --ca-certificates-bundle-s3-key ca-bundle.pem
```

#### 2. Associate with Listener
```bash
$ aws elbv2 modify-listener \
    --listener-arn arn:aws:elasticloadbalancing:region:account:listener/app/my-load-balancer/1234567890123456/1234567890123456 \
    --mutual-authentication Mode=verify,TrustStoreArn=arn:aws:elasticloadbalancing:region:account:truststore/my-trust-store/1234567890123456
```

#### 3. Certificate Format Requirements
- **Format**: PEM format CA certificates
- **Size Limit**: Maximum 1MB
- **Certificate Count**: Maximum 10,000 certificates per trust store
- **Update Method**: Upload new certificate bundle via S3

### Trust Store Best Practices

#### Security Considerations
1. **Regular Certificate Updates**: Establish certificate rotation mechanism
2. **Principle of Least Privilege**: Grant only necessary certificate access permissions
3. **Monitoring and Logging**: Log certificate validation events
4. **Backup Strategy**: Regularly backup certificates and trust store configuration

#### Operations Management
1. **Certificate Lifecycle Management**: Track certificate expiration times
2. **Automated Deployment**: Use Infrastructure as Code for management
3. **Test Environment**: Create test trust store
4. **Error Handling**: Configure logic for certificate validation failures

---

## Load Balancer Advanced Features

### Routing Rules Configuration

#### ALB Routing Rule Types
- **Path Routing**: Distribute traffic based on URL path
- **Host Routing**: Distribute traffic based on Host header
- **Header Routing**: Distribute traffic based on HTTP headers
- **Query String Routing**: Distribute traffic based on query parameters

#### Routing Rule Example
```bash
# Create path routing rule
$ aws elbv2 create-rule \
    --listener-arn arn:aws:elasticloadbalancing:region:account:listener/app/my-load-balancer/1234567890123456/1234567890123456 \
    --conditions Field=path-pattern,Values="/api/*" \
    --priority 100 \
    --actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:region:account:targetgroup/api-targets/1234567890123456
```

### Cross-Zone Load Balancing

#### Cross-Zone Load Balancing Configuration
- **Default Behavior**: ALB and GLB enabled by default, NLB disabled by default
- **Performance Impact**: May increase cross-zone traffic costs
- **High Availability**: Improves fault tolerance capability

#### Cost Optimization Considerations
- **Zone-Aware Routing**: Prioritize routing to targets in same zone
- **Traffic Analysis**: Monitor cross-zone traffic patterns
- **Cost Monitoring**: Set up cross-zone traffic alerts

### AWS Service Integration

#### Auto Scaling Integration
- **Dynamic Scaling**: Automatically adjust target count based on load
- **Health Check Integration**: Automatically replace unhealthy instances
- **Warm-up Time**: New instance preparation time configuration

#### CloudWatch Monitoring
- **Key Metrics**: Request count, latency, error rate
- **Custom Alarms**: Set performance and availability alarms
- **Log Analysis**: Store access logs to S3 for analysis

#### WAF Integration
- **Security Protection**: Web Application Firewall rules
- **DDoS Protection**: Integration with AWS Shield
- **Geographic Blocking**: Control access based on geographic location

---

## Load Balancer Best Practices

### Performance Optimization
1. **Connection Reuse**: Enable Keep-Alive connections
2. **Compression Settings**: Enable HTTP compression
3. **Caching Strategy**: Configure appropriate cache headers
4. **SSL Termination**: Handle SSL at load balancer level

### Security Hardening
1. **SSL/TLS Policies**: Use latest security policies
2. **Access Control**: Configure security group rules
3. **Certificate Management**: Use AWS Certificate Manager
4. **Monitoring Logs**: Enable access log recording

### Troubleshooting
1. **Health Check Failures**: Check target health status
2. **Connection Timeouts**: Adjust timeout settings
3. **502/503 Errors**: Check backend service status
4. **SSL Handshake Failures**: Verify certificate configuration

### Cost Optimization
1. **Load Balancer Type Selection**: Choose appropriate type based on needs
2. **Target Group Configuration**: Optimize target count and distribution
3. **Usage Monitoring**: Regularly check traffic patterns
4. **Reserved Capacity**: Consider using Savings Plans