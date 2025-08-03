# EKS IAM OIDC Architecture Analysis

[English](../en/39_eks_iam_oidc_architecture.md) | [繁體中文](../zh-tw/39_eks_iam_oidc_architecture.md) | [日本語](../ja/39_eks_iam_oidc_architecture.md) | [Back to Index](../README.md)

## Overview

In AWS EKS environments, to enable Kubernetes Service Accounts to securely use AWS services, an identity authentication bridge must be established through the IAM OIDC (OpenID Connect) mechanism. This document uses the AWS Load Balancer Controller as an example to explain the operational principles of this architecture.

## AWS IAM Three-Tier Architecture

AWS IAM uses a three-tier architecture to manage permissions:

1. **Policy**: Permission list (defines what can be done)
2. **Role**: Identity container (defines who can obtain these permissions)
3. **Trust Relationship**: Trust relationship (defines who can assume this role)

## Practical Architecture Example

### Module 2: iam-oidc Module Responsibilities

**Function**: Establish basic permission definitions and authentication mechanisms

```hcl
# Create OIDC Identity Provider
resource "aws_iam_openid_connect_provider" "eks" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [local.oidc_thumbprint]
  url             = var.oidc_issuer_url
}

# Create IAM Policy
resource "aws_iam_policy" "alb_controller" {
  name        = "${local.prefix}-alb-controller-policy"
  description = "IAM policy for AWS Load Balancer Controller"
  policy      = data.http.alb_controller_policy.response_body
}
```

**Key Points**:
- ❌ **Does NOT** create IAM Role
- ❌ **Does NOT** create Kubernetes Service Account
- ✅ **Only creates** IAM Policy (permission list)
- ✅ **Only creates** OIDC Provider (authentication mechanism)

**Policy Source**:
- Downloads standard IAM policy JSON from AWS official GitHub
- URL: `https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.9.0/docs/install/iam_policy.json`
- Contains all AWS API permissions required for ALB/NLB management

### Module 3: lb-ctl Module Responsibilities

**Function**: Create IAM Role and assign permissions to specific identities

```hcl
# Create IAM Role
resource "aws_iam_role" "load_balancer_controller" {
  name = "${local.prefix}-role"

  # OIDC trust relationshop - allow specific service account to assume this role
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = var.oidc_provider_arn # Trusted OIDC provider
        }
        Action = "sts:AssumeRoleWithWebIdentity" # Assume role via OIDC token
        Condition = {
          # Only allow the specific service account to assume this role
          # The following two conditions are required at the same time for assuming this role
          StringEquals = {
            "${replace(data.aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://", "")}:sub" = "system:serviceaccount:kube-system:aws-load-balancer-controller"
            "${replace(data.aws_eks_cluster.main.identity[0].oidc[0].issuer, "https://", "")}:aud" = "sts.amazonaws.com"
          }
        }
      }
    ]
  })
}

# Attach Policy to Role
resource "aws_iam_role_policy_attachment" "load_balancer_controller" {
  policy_arn = var.load_balancer_controller_policy_arn
  role       = aws_iam_role.load_balancer_controller.name
}
```

The Service Account definition is as follows:
```hcl
# Kubernetes Service Account for Load Balancer Controller
resource "kubernetes_service_account" "load_balancer_controller" {
  metadata {
    name      = "aws-load-balancer-controller"
    namespace = "kube-system"

    # Link to the IAM role so the pods can take on the IAM permissions
    annotations = {
      "eks.amazonaws.com/role-arn" = aws_iam_role.load_balancer_controller.arn
    }

    labels = merge(local.common_tags, {
      "app.kubernetes.io/name"       = "aws-load-balancer-controller"
      "app.kubernetes.io/component"  = "controller"
      "app.kubernetes.io/managed-by" = var.managed_by
    })
  }

  depends_on = [aws_iam_role.load_balancer_controller]
}

```
The `metadata.name/metadata.namespace` defined here is the specific service account referenced in the first condition of the `StringEquals` block in the `assume_role_policy` above.

## OIDC Trust Policy Detailed Explanation

### Dual Authentication Mechanism

In the `Condition` of `assume_role_policy`, there are two `StringEquals` conditions:

1. **sub (Subject) Verification**:
   ```
   "${OIDC_ISSUER}:sub" = "system:serviceaccount:kube-system:aws-load-balancer-controller"
   ```
   - Verifies specific Kubernetes Service Account
   - Ensures only the specified Service Account can assume this role

2. **aud (Audience) Verification**:
   ```
   "${OIDC_ISSUER}:aud" = "sts.amazonaws.com"
   ```
   - Verifies the target audience of the OIDC token
   - Ensures the token is issued by AWS STS, preventing token abuse by other services

### Security Principles

Both conditions must be satisfied **simultaneously** to successfully assume the role:
- **Identity Verification**: Confirms it's the correct Service Account
- **Source Verification**: Confirms the token source is legitimate

## Complete Authentication Flow

1. **Kubernetes Service Account** (`aws-load-balancer-controller`)
2. **Via OIDC** requests temporary credentials from AWS STS
3. **STS Verification** of Service Account identity and token source
4. **Assume IAM Role** to obtain permissions
5. **Execute AWS API** operations (create/manage ALB/NLB)

## Module Division Summary

| Module | Responsibility | Created Resources |
|--------|----------------|-------------------|
| iam-oidc | Define permissions | IAM Policy + OIDC Provider |
| lb-ctl | Assign permissions | IAM Role + Role Policy Attachment |

**Design Advantages**:
- **Separation of Concerns**: Permission definition and identity management are separated
- **Reusability**: Policy can be used by multiple Roles
- **Security**: Dual authentication ensures identity security
- **Standardization**: Uses AWS officially recommended Policy content

## Related Concepts

- **OIDC (OpenID Connect)**: Identity authentication protocol
- **STS (Security Token Service)**: AWS temporary credential service
- **Service Account**: Identity identifier for Pods in Kubernetes
- **AssumeRoleWithWebIdentity**: API for obtaining AWS permissions through Web Identity