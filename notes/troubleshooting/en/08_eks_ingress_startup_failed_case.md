# Complete Troubleshooting Case: EKS Ingress Startup Failure

[English](../en/08_eks_ingress_startup_failed_case.md) | [繁體中文](../zh-tw/08_eks_ingress_startup_failed_case.md) | [日本語](../ja/08_eks_ingress_startup_failed_case.md) | [Back to Index](../README.md)

---

## Background
- Experiment Date: 2025/08/10
- Difficulty: 🤬🤬🤬🤬
- Description: After deploying an EKS cluster, the ALB Ingress Controller cannot create an Application Load Balancer, and the ADDRESS field of the Ingress resource remains blank.

---

## Initial Symptoms

### Application Status
```bash
$ kubectl get pods -n nebuletta-app-ns
NAME                                   READY   STATUS    RESTARTS   AGE
beta-app-deployment-7d7575984d-48jwp   1/1     Running   0          12m
beta-app-deployment-7d7575984d-g7bth   1/1     Running   0          12m
beta-app-deployment-7d7575984d-s695t   1/1     Running   0          12m
prod-app-deployment-68cc5659d9-2kb4t   1/1     Running   0          12m
prod-app-deployment-68cc5659d9-4w2gm   1/1     Running   0          12m
prod-app-deployment-68cc5659d9-crft7   1/1     Running   0          12m

$ kubectl get svc -n nebuletta-app-ns
NAME                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
beta-app-service-clusterip   ClusterIP   172.20.94.217   <none>        8080/TCP   11m
prod-app-service-clusterip   ClusterIP   172.20.23.79    <none>        8080/TCP   11m

$ kubectl get ingress -n nebuletta-app-ns
NAME           CLASS   HOSTS                   ADDRESS   PORTS   AGE
ingress-path   alb     eks-lab.nebuletta.com             80      11m
```

**Key Issue**: The ADDRESS field of the Ingress is blank, indicating that the ALB was not successfully created.

---

## Troubleshooting Process

### Phase 1: Initial Error Analysis

First, update the local kubectl to ensure we're connecting to the correct cluster:
```bash
# Adjust region and cluster name according to your AWS configuration
$ aws eks update-kubeconfig --region ap-northeast-1 --name dev-eks-cluster
```

Check the Load Balancer Controller logs:

```bash
$ kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller --tail=50
```

**Error Found**:
```
{"level":"error","ts":"2025-08-10T06:31:17Z","msg":"Reconciler error","controller":"ingress","object":{"name":"nebuletta-eks-lab"},"namespace":"","name":"nebuletta-eks-lab","reconcileID":"adf58942-8802-4c8c-9063-d6c3dea9d5c9","error":"WebIdentityErr: failed to retrieve credentials\ncaused by: RequestError: send request failed\ncaused by: Post \"https://sts.ap-northeast-1.amazonaws.com/\": dial tcp: lookup sts.ap-northeast-1.amazonaws.com on 172.20.0.10:53: read udp 10.0.12.84:49309->172.20.0.10:53: read: connection refused"}
```

**Analysis**: The Load Balancer Controller cannot connect to AWS STS service to retrieve credentials.

### Phase 2: Check VPC Endpoints

Check if the STS VPC Endpoint exists:

```bash
$ aws ec2 describe-vpc-endpoints --region ap-northeast-1 --filters "Name=tag:Name,Values=dev-networking-sts-endpoint" | jq

{
  "VpcEndpoints": [
    {
      "VpcEndpointId": "vpce-0cb49d2450a183b3f",
      "VpcEndpointType": "Interface",
      "VpcId": "vpc-0ef535df4de8e80bb",
      "ServiceName": "com.amazonaws.ap-northeast-1.sts",
      "State": "available",
      "PrivateDnsEnabled": true,
      // ... other details
    }
  ]
}
```

**Result**: The STS VPC Endpoint exists and is in `available` state, but still cannot connect.

### Phase 3: Deep DNS Issue Analysis

Test DNS resolution:

```bash
$ kubectl run dns-test --image=busybox:1.28 --rm -it --restart=Never -- nslookup sts.ap-northeast-1.amazonaws.com
# DNS query hangs with no response
```

Check CoreDNS status:

```bash
$ kubectl get pods -n kube-system -l k8s-app=kube-dns
NAME                       READY   STATUS    RESTARTS   AGE
coredns-5c658475b5-4hkn8   0/1     Pending   0          53m
coredns-5c658475b5-pcjxv   0/1     Pending   0          53m
```

**Root Cause Discovered**: CoreDNS pods are in `Pending` state, causing the entire cluster to be unable to perform DNS resolution!

### Phase 4: CoreDNS Scheduling Issue Analysis

Check CoreDNS pods detailed status:

```bash
$ kubectl describe pods -n kube-system -l k8s-app=kube-dns
```

**Key Error Message**:
```
Events:
  Type     Reason            Age                    From               Message
  ----     ------            ----                   ----               -------
  Warning  FailedScheduling  14m (x4 over 30m)      default-scheduler  0/8 nodes are available: 8 node(s) had untolerated taint {eks.amazonaws.com/compute-type: fargate}. preemption: 0/8 nodes are available: 8 Preemption is not helpful for scheduling.
```

**Problem Cause**:
- All nodes are Fargate nodes with the taint `eks.amazonaws.com/compute-type: fargate`
- CoreDNS pods lack the corresponding toleration and cannot be scheduled on Fargate nodes

Check Fargate Profile configuration:

```bash
$ aws eks describe-fargate-profile --region ap-northeast-1 --cluster-name dev-eks-cluster --fargate-profile-name dev-eks-cluster-fp-default | jq
```

**Confirmed**: The Fargate profile does include the `kube-system` namespace, but CoreDNS still cannot be scheduled.

---

## Root Cause

### Main Problem Chain
1. **CoreDNS lacks Fargate toleration** → CoreDNS pods cannot be scheduled
2. **DNS resolution failure** → All DNS queries fail  
3. **Load Balancer Controller cannot connect to STS** → Cannot obtain AWS credentials
4. **ALB cannot be created** → Ingress ADDRESS field is blank
5. **Application cannot be accessed through Ingress** → Only port-forward works

### Technical Principles
- In **EKS Fargate** environment, all pods run on Fargate nodes
- **Fargate nodes** have a special taint: `eks.amazonaws.com/compute-type: fargate`
- **System components** (like CoreDNS) don't have corresponding tolerations by default
- **Stateless DNS** issues cause the entire service chain to fail

---

## Solutions

### Solution Step 1: Fix CoreDNS Toleration

Add Fargate toleration to the CoreDNS deployment:

```bash
$ kubectl patch deployment coredns -n kube-system --type='merge' -p='
{
  "spec": {
    "template": {
      "spec": {
        "tolerations": [
          {
            "key": "CriticalAddonsOnly",
            "operator": "Exists"
          },
          {
            "key": "node-role.kubernetes.io/control-plane",
            "effect": "NoSchedule"
          },
          {
            "key": "node.kubernetes.io/not-ready",
            "effect": "NoExecute",
            "tolerationSeconds": 300
          },
          {
            "key": "node.kubernetes.io/unreachable",
            "effect": "NoExecute", 
            "tolerationSeconds": 300
          },
          {
            "key": "eks.amazonaws.com/compute-type",
            "operator": "Equal",
            "value": "fargate",
            "effect": "NoSchedule"
          }
        ]
      }
    }
  }
}'

deployment.apps/coredns patched
```

**Verify Fix**:
```bash
$ kubectl get pods -n kube-system -l k8s-app=kube-dns
NAME                      READY   STATUS    RESTARTS   AGE
coredns-56ffbd694-6xv6f   1/1     Running   0          71s
coredns-56ffbd694-ppz46   1/1     Running   0          71s
```

### Solution Step 2: Discover New Issue - Missing Subnet Tags

After CoreDNS was fixed, the Load Balancer Controller showed a new error:

```bash
$ kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller --tail=20
```

**New Error Message**:
```
{"level":"error","ts":"2025-08-10T07:04:02Z","msg":"Reconciler error","controller":"ingress","object":{"name":"nebuletta-eks-lab"},"namespace":"","name":"nebuletta-eks-lab","reconcileID":"e4c5afd6-fb56-495e-9a7c-30c7450e2ecc","error":"couldn't auto-discover subnets: unable to resolve at least one subnet (0 match VPC and tags: [kubernetes.io/role/elb])"}
```

**Problem Analysis**: The Load Balancer Controller cannot find appropriate subnets to deploy the ALB.

Check subnet tags:

```bash
# Check public subnets
$ aws ec2 describe-subnets --region ap-northeast-1 --filters "Name=vpc-id,Values=vpc-0ef535df4de8e80bb" --query 'Subnets[?MapPublicIpOnLaunch==`true`].[SubnetId,Tags]'

# Check private subnets  
$ aws ec2 describe-subnets --region ap-northeast-1 --filters "Name=vpc-id,Values=vpc-0ef535df4de8e80bb" --query 'Subnets[?MapPublicIpOnLaunch==`false`].[SubnetId,Tags]'
```

**Issue Found**: Subnets are missing Kubernetes tags required by ALB:
- Public subnets missing: `kubernetes.io/role/elb = 1`
- Private subnets missing: `kubernetes.io/role/internal-elb = 1`

### Solution Step 3: Add Subnet Tags

Manually add the missing tags:

```bash
# Add tags to public subnets
$ aws ec2 create-tags --region ap-northeast-1 --resources subnet-03137f56962d0a148 subnet-0a11863cea526b5a0 subnet-00b4e519a524eb1f3 --tags Key=kubernetes.io/role/elb,Value=1

# Add tags to private subnets
$ aws ec2 create-tags --region ap-northeast-1 --resources subnet-00d1d5009e0fd2145 subnet-0dbcd8d5c45e993f5 subnet-087c8cf6e4ba0d2ae --tags Key=kubernetes.io/role/internal-elb,Value=1
```

### Solution Result Verification

After adding the tags, the Load Balancer Controller successfully created the ALB:

```bash
$ kubectl get ingress -n nebuletta-app-ns
NAME           CLASS   HOSTS                   ADDRESS                                                                      PORTS   AGE
ingress-path   alb     eks-lab.nebuletta.com   k8s-nebulettaekslab-4b28eb94af-1752493976.ap-northeast-1.elb.amazonaws.com   80      42m
```

**Success Logs**:
```
{"level":"info","ts":"2025-08-10T07:09:35Z","logger":"controllers.ingress","msg":"created loadBalancer","stackID":"nebuletta-eks-lab","resourceID":"LoadBalancer","arn":"arn:aws:elasticloadbalancing:ap-northeast-1:362395300803:loadbalancer/app/k8s-nebulettaekslab-4b28eb94af/3e2e73703b34a339"}
{"level":"info","ts":"2025-08-10T07:09:35Z","logger":"controllers.ingress","msg":"created listener","stackID":"nebuletta-eks-lab","resourceID":"80","arn":"arn:aws:elasticloadbalancing:ap-northeast-1:362395300803:listener/app/k8s-nebulettaekslab-4b28eb94af/3e2e73703b34a339/d26a631eb694bf64"}
{"level":"info","ts":"2025-08-10T07:09:36Z","logger":"controllers.ingress","msg":"successfully deployed model","ingressGroup":"nebuletta-eks-lab"}
```

**Application Testing**:
```bash
$ curl k8s-nebulettaekslab-4b28eb94af-1752493976.ap-northeast-1.elb.amazonaws.com:80/beta -H 'Host: eks-lab.nebuletta.com'
[beta] served by: beta-app-deployment-7d7575984d-48jwp

$ curl k8s-nebulettaekslab-4b28eb94af-1752493976.ap-northeast-1.elb.amazonaws.com:80/prod -H 'Host: eks-lab.nebuletta.com'
[prod] served by: prod-app-deployment-68cc5659d9-4w2gm
```

---

## Terraform Module Improvement Suggestions

### 1. Use EKS Addons to Manage System Components (Recommended)

#### Best Practice: Official EKS Addons

After actual verification, the best solution is to use AWS EKS's official addons to manage the system rather than manually patching Kubernetes resources. This approach is more stable, reliable, and fully managed by AWS.

**Add `terraform/modules/eks-lab/cluster/addons.tf`**:
```hcl
# EKS Addons Configuration - Manage basic system components
# These addons automatically handle Fargate compatibility issues

# VPC CNI addon - Handle Pod networking
resource "aws_eks_addon" "vpc_cni" {
  cluster_name                = aws_eks_cluster.main.name
  addon_name                  = "vpc-cni"
  addon_version              = var.vpc_cni_addon_version
  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "PRESERVE"

  depends_on = [
    aws_eks_cluster.main,
    aws_eks_fargate_profile.default
  ]
}

# Kube-proxy addon - Handle service networking
resource "aws_eks_addon" "kube_proxy" {
  cluster_name                = aws_eks_cluster.main.name
  addon_name                  = "kube-proxy"
  addon_version              = var.kube_proxy_addon_version
  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "PRESERVE"

  depends_on = [
    aws_eks_cluster.main,
    aws_eks_fargate_profile.default
  ]
}

# CoreDNS addon - Auto-configure Fargate toleration
resource "aws_eks_addon" "coredns" {
  cluster_name                = aws_eks_cluster.main.name
  addon_name                  = "coredns"
  addon_version              = var.coredns_addon_version
  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "PRESERVE"

  configuration_values = jsonencode({
    tolerations = [
      {
        key      = "CriticalAddonsOnly"
        operator = "Exists"
      },
      {
        key    = "node-role.kubernetes.io/control-plane"
        effect = "NoSchedule"
      },
      {
        key               = "node.kubernetes.io/not-ready"
        effect            = "NoExecute"
        tolerationSeconds = 300
      },
      {
        key               = "node.kubernetes.io/unreachable"
        effect            = "NoExecute"
        tolerationSeconds = 300
      },
      {
        key      = "eks.amazonaws.com/compute-type"
        operator = "Equal"
        value    = "fargate"
        effect   = "NoSchedule"
      }
    ]
  })

  depends_on = [
    aws_eks_cluster.main,
    aws_eks_fargate_profile.default
  ]
}

# Metrics Server addon - Support HPA and other features
resource "aws_eks_addon" "metrics_server" {
  cluster_name                = aws_eks_cluster.main.name
  addon_name                  = "metrics-server"
  addon_version              = var.metrics_server_addon_version
  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "PRESERVE"

  configuration_values = jsonencode({
    tolerations = [
      {
        key      = "CriticalAddonsOnly"
        operator = "Exists"
      },
      {
        key    = "node-role.kubernetes.io/control-plane"
        effect = "NoSchedule"
      },
      {
        key               = "node.kubernetes.io/not-ready"
        effect            = "NoExecute"
        tolerationSeconds = 300
      },
      {
        key               = "node.kubernetes.io/unreachable"
        effect            = "NoExecute"
        tolerationSeconds = 300
      },
      {
        key      = "eks.amazonaws.com/compute-type"
        operator = "Equal"
        value    = "fargate"
        effect   = "NoSchedule"
      }
    ]
  })

  depends_on = [
    aws_eks_cluster.main,
    aws_eks_fargate_profile.default
  ]
}
```

**Modify `terraform/modules/eks-lab/cluster/variables.tf`**, add version variables:
```hcl
# EKS Addons version variables, you can check versions on AWS EKS addons console
variable "vpc_cni_addon_version" {
  description = "VPC CNI addon version"
  type        = string
  default     = "v1.19.0-eksbuild.1"
}

variable "kube_proxy_addon_version" {
  description = "Kube-proxy addon version"
  type        = string
  default     = "v1.33.0-eksbuild.2"
}

variable "coredns_addon_version" {
  description = "CoreDNS addon version"
  type        = string
  default     = "v1.11.4-eksbuild.2"
}

variable "metrics_server_addon_version" {
  description = "Metrics Server addon version"
  type        = string
  default     = "v0.8.0-eksbuild.1"
}
```

#### Why Use EKS Addons?

1. **Official Support**: Directly maintained and managed by AWS
2. **Automatic Updates**: Can be configured for automatic updates to compatible versions
3. **Fargate Compatibility**: Automatically handle toleration and scheduling requirements
4. **Consistency**: Avoid configuration drift that may occur with manual patching
5. **Reliability**: Reduces complexity of cross-provider operations

#### Comparison with Old Methods

| Method | Manual kubernetes_manifest Patching | EKS Addons |
|--------|-------------------------------------|------------|
| Complexity | High (requires cross-provider operations) | Low (AWS native management) |
| Reliability | Medium (depends on execution order) | High (AWS guarantee) |
| Maintainability | Difficult (requires manual updates) | Simple (can auto-update) |
| Error Handling | Complex | Built-in conflict resolution |

### 2. Subnet Tags Fix

Modify the `networking` module's `vpc.tf`:

```hcl
# Public subnets
resource "aws_subnet" "public" {
  count                   = length(local.public_subnet_cidrs)
  vpc_id                  = aws_vpc.this.id
  cidr_block              = local.public_subnet_cidrs[count.index]
  availability_zone       = local.available_azs[count.index]
  map_public_ip_on_launch = true
  
  tags = merge(local.common_tags, {
    Name                     = "${local.prefix}-public-${count.index}"
    "kubernetes.io/role/elb" = "1"  # Add this tag
  })
}

# Private subnets
resource "aws_subnet" "private" {
  count             = length(local.private_subnet_cidrs)
  vpc_id            = aws_vpc.this.id
  cidr_block        = local.private_subnet_cidrs[count.index]
  availability_zone = local.available_azs[count.index]
  
  tags = merge(local.common_tags, {
    Name                              = "${local.prefix}-private-${count.index}"
    "kubernetes.io/role/internal-elb" = "1"  # Add this tag
  })
}
```

#### Why Must the Tag Value Be "1"?

This is an important detail that many people wonder about - why not `"true"` or other values:

**1. AWS Official Documentation Requirement**
- AWS Load Balancer Controller official documentation explicitly states that the tag value must be `"1"`
- This is a standard convention in the AWS and Kubernetes ecosystem

**2. Tag Search Mechanism**
```bash
# AWS Load Balancer Controller internally executes searches similar to:
aws ec2 describe-subnets --filters "Name=tag:kubernetes.io/role/elb,Values=1"
aws ec2 describe-subnets --filters "Name=tag:kubernetes.io/role/internal-elb,Values=1"
```

**3. Value Comparison**
| Tag Value | Support Status | Description |
|----------|---------------|-------------|
| `"1"` | ✅ Official Standard | Value required by AWS official documentation |
| `"true"` | ❌ Not Supported | Though semantically correct, the Controller won't recognize it |
| `"yes"` | ❌ Not Supported | Similarly not recognized by the Controller |
| `""` (empty value) | ⚠️ Partial Support | Technically valid but not recommended |

**4. Historical Background**
- This convention originated from early Kubernetes label systems
- In boolean marking, `"1"` represents "enabled" or "true"
- Maintained for backward compatibility and continues to this day

**5. Practical Verification**
If you use incorrect tag values, the Load Balancer Controller will report an error:
```
couldn't auto-discover subnets: unable to resolve at least one subnet (0 match VPC and tags: [kubernetes.io/role/elb])
```

**Remember**: You must use `"1"` - this is not an arbitrary choice but an AWS hard requirement!

---

## Key Learning Points

### EKS Fargate Specifics

1. **Taint Mechanism**: Fargate nodes have special taints, requiring corresponding tolerations
2. **System Components**: CoreDNS and other system components may need additional configuration to run on Fargate
3. **DNS Dependency**: The entire cluster's network functionality depends on CoreDNS running properly

### ALB Ingress Controller Requirements

1. **Subnet Tags**: Must correctly tag subnets for their purpose
   - `kubernetes.io/role/elb`: For public ALB
   - `kubernetes.io/role/internal-elb`: For internal ALB
2. **Network Connectivity**: Must be able to access AWS APIs (through VPC endpoints or NAT Gateway)
3. **IAM Permissions**: Requires IRSA with correct IAM permissions

### Problem Diagnosis Techniques

1. **Outside-in**: Start troubleshooting from user-visible symptoms
2. **Check Dependency Chain**: Confirm that each component's dependent services are functioning
3. **Layered Diagnosis**: Check systematically from bottom layer (DNS) to top layer (application)
4. **Log Analysis**: Use kubectl logs effectively to find specific error messages

### Additional Notes
- Actually, in the [EKS Cluster Creation](../../containerization/orchestration/Kubernetes/en/35_eks_practice.md#eks-cluster-creation) section of the AWS EKS Practical Guide, you can see the following logs:
  ```bash
  default addons vpc-cni, kube-proxy, coredns, metrics-server were not specified, will install them as EKS addons
  ...
  creating addon: coredns
  successfully created addon: coredns
  ...
  "coredns" is now schedulable onto Fargate
  "coredns" is now scheduled onto Fargate
  "coredns" pods are now scheduled onto Fargate
  ```
- This indicates:
  1. EKS Addon Management: eksctl uses the EKS addon system to deploy CoreDNS, rather than a simple Kubernetes Deployment
  2. Fargate Awareness: When using `eksctl create cluster --fargate`, eksctl will:
    - Automatically create a default Fargate Profile
    - Automatically configure CoreDNS to be schedulable on Fargate
    - Handle necessary toleration settings
  3. Difference from Terraform: What we create with Terraform is:
    - A native EKS cluster
    - Manually created Fargate Profile
    - CoreDNS is still the EKS default deployment, but without automatic Fargate scheduling configuration
- This is why we didn't encounter this issue when manually operating through eksctl, because AWS handled it proactively

---

## Preventive Measures

1. **Module Design**: Consider EKS-specific requirements in infrastructure modules from the start
2. **Automated Testing**: Automatically verify DNS resolution and basic functionality after deployment
3. **Monitoring and Alerting**: Set up health checks for CoreDNS and Load Balancer Controller
4. **Documentation**: Record known issues and solutions to avoid repeated pitfalls

---

## Important Note: Terraform Output and ALB Readiness Timing

### Asynchronous Nature of ALB Creation

After deploying the applications stack, you may encounter the following situation:

```bash
# kubectl shows ingress has ADDRESS
$ kubectl get ingress -n nebuletta-app-ns
NAME           CLASS   HOSTS                   ADDRESS
ingress-path   alb     eks-lab.nebuletta.com   k8s-nebulettaekslab-4b28eb94af-1249260893.ap-northeast-1.elb.amazonaws.com

# But terraform output shows empty values
$ terramate run --tags dev-eks-applications -- terraform output
ingress_address = ""
ingress_hostname = ""
```

### Cause Analysis

This occurs because:
1. **ALB creation is asynchronous**: When Terraform apply completes, the ALB may still be in the process of being created
2. **Terraform state snapshot**: Output shows values from the last time resource state was read
3. **Kubernetes real-time state**: kubectl shows the cluster's real-time state

### Solutions

**Method 1: Refresh Terraform State (Recommended)**
```bash
# Refresh state to get latest ingress information
terramate run --tags dev-eks-applications -- terraform refresh

# Check output again
terramate run --tags dev-eks-applications -- terraform output
```

**Method 2: Re-run Apply**
```bash
# Re-running apply will automatically refresh state
terramate run --tags dev-eks-applications -- terraform apply -auto-approve

# Check output
terramate run --tags dev-eks-applications -- terraform output
```

**Method 3: Continuous Monitoring**
```bash
# Check every 30 seconds
watch -n 30 "terramate run --tags dev-eks-applications -- terraform output ingress_address"
```

### Time Expectations

- **ALB Creation Time**: Usually takes 2-5 minutes
- **DNS Propagation Time**: Additional 1-2 minutes
- **Total Wait Time**: Approximately 3-7 minutes

### Verify ALB Ready State

```bash
# Method 1: Check Kubernetes ingress status
kubectl get ingress -n nebuletta-app-ns

# Method 2: Check AWS Load Balancer Controller logs
kubectl logs -n kube-system deployment/aws-load-balancer-controller

# Method 3: Test DNS resolution
dig +short k8s-nebulettaekslab-4b28eb94af-1249260893.ap-northeast-1.elb.amazonaws.com
```

### Reminder

**Don't rush to troubleshoot**: If kubectl shows the ingress has an ADDRESS but terraform output is empty, **first wait for the ALB to be fully ready**, then run `terraform refresh`. This is normal asynchronous behavior, not a configuration error.

---

## Conclusion

This troubleshooting experience demonstrates the complexity of the EKS Fargate environment. What seemed like a simple "Ingress cannot create ALB" problem actually involved:

1. **CoreDNS scheduling issues** (Fargate toleration)
2. **DNS resolution failures** (entire cluster network functionality interrupted)
3. **AWS API access issues** (STS connection failure)
4. **Missing subnet tags** (ALB cannot find appropriate subnets)

**Key Point**: In EKS Fargate environments, correct configuration of system components is the foundation of the entire cluster's functionality. CoreDNS, as the core of DNS services, will cause a chain reaction when it fails, affecting all services that need to perform DNS queries. The symptoms of problems are often not the root cause, requiring systematic analysis to find the real underlying issues.
