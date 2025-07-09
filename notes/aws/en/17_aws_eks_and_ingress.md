# AWS EKS and Ingress Integration

[English](../en/17_aws_eks_and_ingress.md) | [繁體中文](../zh-tw/17_aws_eks_and_ingress.md) | [日本語](../ja/17_aws_eks_and_ingress.md) | [Back to Index](../README.md)

## Architecture Overview

```
              Admin EC2                                   Cluster                                                           
  +-------------------------------+    +------------------------------------------------+                                   
  | +--------+                    |    |  +----Service A----+       +---Service B----+  |                                   
  | | eksctl |                    |    |  | +-----+ +-----+ |       |+-----+ +-----+ |  |                                   
  | +--------+                    |    |  | | Pod | | Pod | |-------|| Pod | | Pod | |  |                                   
  |  |                            |    |  | +-----+ +-----+ |   |   |+-----+ +-----+ |  |                                   
  |  |--create k8s cluster ----------->|  +-----------------+   |   +----------------+  |                                   
  |  |                            |    |                        |                       |                                   
  |  +--create cluster resources ----->|            +--------------------+              |                                   
  |                               |    |            |       Ingress      |              |              +---------+          
  | +---------+                   |    |            +--------------------+              |          ----| AWS ALB |          
  | | kubectl |                   |    |                        |                       |    -----/    +---------+          
  | +---------+                   |    |           +------------------------+          -----/               ^               
  |  |                            |    |           | +--------------------+ |    -----/ |                   |               
  |  |--access k8s cluster ----------->|           | | AWS Load Balancer  | |---/       |                   |  able to create               
  |                               |    |           | | Ingress Controller | |           |                   |               
  +-------------------------------+    |           | +--------------------+ |           |              +----------+         
                                       |           | +--------------------+ |           |              | IAM ROLE |         
                                       |           | |   Service Account  | |           |              +----------+         
                                       |           | +--------------------+ |           |                   |               
                                       |           +------------------------+           |                   |               
                                       |                        |                       |                   |               
                                       |         +-----------------------------+        |        +-----------------------+  
                                       |         | OpenID Connect Provider URL |-----------------| IAM Identity Provider |  
                                       |         +-----------------------------+        |        +-----------------------+  
                                       +------------------------------------------------+                                   
```

## Architecture Components

### Management Layer (Admin EC2)

#### eksctl Tool
- **Purpose**: Create and manage EKS clusters
- **Main Functions**:
  - Create Kubernetes clusters
  - Configure cluster resources (IAM roles, security groups, etc.)
  - Manage node groups

#### kubectl Tool
- **Purpose**: Kubernetes cluster management tool
- **Main Functions**:
  - Access and manage resources within the cluster
  - Deploy applications
  - Monitor cluster status

### Inside EKS Cluster

#### Application Service Layer
- **Service A/B**: Services containing multiple Pods
- **Load Balancing**: Internal load balancing using Kubernetes mechanisms

#### Ingress Resource
- **Function**: Defines how external traffic enters the cluster
- **Features**: Routing rule configuration, SSL/TLS termination, host and path-based routing

## Permission Management Architecture

### Key Difference Between AWS EKS and Local K8s

#### OpenID Connect Provider URL
- EKS automatically generates OIDC Provider URL
- Provides K8s Cluster with recognizable "identity," enabling components within K8s to request permissions (IAM Role) from AWS

#### IAM Identity Provider + Role
1. **Create IAM Identity Provider**: Bind to EKS's OIDC Provider
2. **Create IAM Role**: Authorize this Role to create Load Balancer (ALB)
3. **Bind Service Account**: Allow Ingress Controller to assume this Role and obtain ALB creation permissions

## Ingress Controller Operation Mechanism

### AWS Load Balancer Controller
- **Deployment Method**: Deploy AWS Load Balancer Ingress Controller inside K8s
- **Monitoring Target**: This Controller monitors K8s Ingress resources
- **Automation**: When creating/updating Ingress, Controller automatically creates ALB in AWS

### Service Account and IAM Role Integration
- Ingress Controller needs permissions to operate AWS resources
- Through Service Account combined with IAM Role
- Provides secure permission management mechanism

## Traffic Flow

### External Traffic Entry Process
1. **User Request** → AWS ALB
2. **ALB Routing** → Routes to corresponding K8s Service based on Ingress rules (Host/Path)
3. **Service Load Balancing** → Target Pod

### ALB Characteristics
- **Layer**: L7 (Application Layer)
- **Features**: Can perform Host/Path traffic distribution
- **Advantage**: More flexibility than NLB

## Service Account Detailed Explanation

### What is a Service Account?
- **Definition**: Kubernetes "machine account"
- **Purpose**: Provides applications inside Pods with their own "identity"
- **Default**: Each Pod has a default Service Account

### Main Uses
- **Control Pod Access**: Manage which K8s APIs Pods can access
- **Permission Separation**: For security, separate different permissions

### Relationship with AWS IAM

#### Binding Mechanism
- Service Account can be bound to AWS IAM Role
- After Pod starts with specific SA, it automatically has corresponding IAM Role permissions
- **Advantage**: No need to write Access Key/Secret Key in code or environment variables

#### Actual Binding Process
1. **Create IAM Role**: Trust EKS OIDC Provider
2. **Create Service Account**: Add annotation in metadata to bind IAM Role ARN
3. **Automatic Credential Issuance**: AWS automatically issues temporary credentials to Pod

#### Example Configuration
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: alb-ingress-controller
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/ALBIngressControllerRole
```

### Key Concept Summary
- **Service Account** = "Machine account" in K8s
- **IAM Role** = Authorization role for AWS services
- **Combination** = Applications on K8s can securely obtain AWS resource operation permissions

## AWS Load Balancer Controller Operation Details

### Controller Itself is Also a Pod
- **Deployment Method**: K8s Deployment, containing one or more Pods
- **Permission Requirements**: Needs permissions to operate AWS ALB resources
- **Solution**: Bind custom Service Account and IAM Role

### Complete Operation Flow
1. **Monitor Changes**: Controller detects Ingress resource changes
2. **Create Resources**: Through IAM Role permissions, automatically create/update ALB on AWS
3. **Traffic Routing**: ALB receives traffic and distributes to different Services based on Ingress rules

## Summary

### Main Differences from Local K8s
- **Permission Integration**: All cross-service resources (like ALB) must complete IAM setup first
- **Automation Level**: K8s Ingress Controller can automatically create and manage AWS ALB
- **Advantages**: Enjoy high availability and operational advantages of ALB

### Core Value
- **Security**: Manage permissions through IAM roles, avoid hardcoded credentials
- **Automation**: Reduce manual configuration work
- **Integration**: Deep integration with AWS services