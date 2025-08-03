# Terraform Provider Scope "no client config" Error

[English](../en/07_terraform_provider_scope_issue.md) | [繁體中文](../zh-tw/07_terraform_provider_scope_issue.md) | [日本語](../ja/07_terraform_provider_scope_issue.md) | [Back to Index](../README.md)

---

## Background
- Experiment Date: 2025/08/03
- Difficulty: 🤬
- Description: Encountered "cannot create REST client: no client config" error when attempting to extract Kubernetes provider configuration into a shared module.

---

## Problem Description

When designing Terraform architecture to avoid duplicating Kubernetes provider configuration across multiple stacks, attempted to create a shared provider module but encountered issues with Kubernetes resources failing to initialize properly.

### Error Message Example

```
╷
│ Error: Failed to construct REST client
│
│   with module.eks_storage.kubernetes_manifest.efs_csi_driver,
│   on ../../../../modules/eks-lab/storage/efs-csi.tf line 160, in resource "kubernetes_manifest" "efs_csi_driver":
│  160: resource "kubernetes_manifest" "efs_csi_driver" {
│
│ cannot create REST client: no client config
╵
```

### Incorrect Architecture Design

```hcl
# ❌ Wrong: Attempting to place provider configuration in a child module
module "eks_providers" {
  source = "../../../../modules/shared/eks-providers"
  
  cluster_name           = data.terraform_remote_state.eks_cluster.outputs.cluster_name
  cluster_endpoint       = data.terraform_remote_state.eks_cluster.outputs.cluster_endpoint
  cluster_ca_certificate = data.terraform_remote_state.eks_cluster.outputs.cluster_certificate_authority_data
  aws_region            = "ap-northeast-1"
}

module "eks_storage" {
  source = "../../../../modules/eks-lab/storage"
  # ❌ This module cannot use the provider configuration from the above child module
  depends_on = [module.eks_providers]
}
```

---

## Root Causes

### 1. Terraform Provider Scope Rules

Terraform's provider configuration has strict scope limitations:

- **Module Internal Isolation**: Each module has its own provider scope
- **Downward Inheritance**: Parent-level provider configurations are automatically passed to child modules
- **No Lateral Sharing**: Provider configurations in child or parallel modules do not affect each other

### 2. Misunderstanding `depends_on` Function

```
❌ Wrong Assumption: depends_on = [module.eks_providers] enables configuration sharing
✅ Reality: depends_on only controls execution order, does not affect provider scope
```

### 3. Provider Initialization Flow

```
1. Terraform reads kubernetes_manifest resource
2. Searches for kubernetes provider configuration in current scope
3. Configuration not found → "cannot create REST client: no client config"
4. Even with depends_on dependency, cannot access other modules' providers
```

---

## Resolution Steps

### Solution 1: Direct Provider Configuration at Stack Level

```hcl
# ✅ Correct: Configure provider directly at stack level
provider "kubernetes" {
  host                   = data.terraform_remote_state.eks_cluster.outputs.cluster_endpoint
  cluster_ca_certificate = base64decode(data.terraform_remote_state.eks_cluster.outputs.cluster_certificate_authority_data)
  
  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "aws"
    args = [
      "eks",
      "get-token",
      "--cluster-name",
      data.terraform_remote_state.eks_cluster.outputs.cluster_name,
      "--region",
      "ap-northeast-1"
    ]
  }
}

module "eks_storage" {
  source = "../../../../modules/eks-lab/storage"
  # ✅ Module automatically inherits provider configuration from same level
}
```

### Solution 2: Use Terramate Shared Templates

#### Step 1: Create Shared Template

Create file `stacks/dev/shared/terramate-templates/eks-k8s-provider.tm.hcl`:

```hcl
generate_hcl "_terramate_generated_k8s_provider.tf" {
  content {
    provider "kubernetes" {
      host                   = data.terraform_remote_state.eks_cluster.outputs.cluster_endpoint
      cluster_ca_certificate = base64decode(data.terraform_remote_state.eks_cluster.outputs.cluster_certificate_authority_data)

      exec {
        api_version = "client.authentication.k8s.io/v1beta1"
        command     = "aws"
        args = [
          "eks",
          "get-token",
          "--cluster-name",
          data.terraform_remote_state.eks_cluster.outputs.cluster_name,
          "--region",
          global.aws_region
        ]
      }
    }
  }
}
```

#### Step 2: Import in Required Stacks

Add to `stack.tm.hcl`:

```hcl
# Import shared provider template
import {
  source = "../../shared/terramate-templates/eks-k8s-provider.tm.hcl"
}
```

#### Step 3: Execute Terramate Generation

```bash
$ terramate generate
```

---

## Technical Principles

### Provider Scope Rules Diagram

```
┌──────────────────────────────────────────────────┐
│                Stack Level                       │
│  ┌─────────────────────────────────────────────┐ │
│  │  provider "kubernetes" { ... }              │ │
│  └─────────────────────┬───────────────────────┘ │
│                        │ (Downward Inheritance)  │
│  ┌─────────────────────▼───────────────────────┐ │
│  │           module "business_logic"           │ │
│  │  ✅ Can use parent provider configuration   │ │
│  └─────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│                Stack Level                       │
│  ┌─────────────────────┐ ┌─────────────────────┐ │
│  │  module "providers" │ │  module "business"  │ │
│  │  Contains provider  │ │  ❌ Cannot access   │ │
│  └─────────────────────┘ └─────────────────────┘ │
│              │                       ▲           │
│              └───── depends_on ──────┘           │
│                   (Controls order only)          │
└──────────────────────────────────────────────────┘
```

### Kubernetes Provider Initialization Flow

```
1. Terraform scans required_providers
   ↓
2. Searches for corresponding provider "kubernetes" configuration
   ↓
3. Uses host, exec parameters to create HTTP client
   ↓ 
4. Calls aws eks get-token to obtain JWT token
   ↓
5. Adds token to Authorization header
   ↓
6. Sends request to EKS API server
```

If step 2 cannot find provider configuration, it generates "no client config" error.

---

## Prevention Measures

### 1. Understand Provider Scope

- ✅ **Correct**: Configure provider at the level where it's needed
- ❌ **Wrong**: Expect provider sharing through child modules or depends_on

### 2. Choose Appropriate Sharing Strategy

| Solution | Advantages | Disadvantages | Use Case |
|----------|------------|---------------|----------|
| Stack-level configuration | Simple and direct | Potential duplication | Single project |
| Terramate templates | Avoid duplication, unified management | Requires tool support | Multi-environment projects |
| Global provider | Complete sharing | Lacks flexibility | Standardized environments |

### 3. Design Principles

- **Provider Proximity Principle**: Configure at the level closest to usage
- **Avoid Inter-module Dependencies**: Don't expect modules to share providers
- **Document Configuration**: Clearly document each provider's configuration source

---

## Related Concepts

### Terraform Module System

```bash
# View module dependency tree
$ terraform graph

# Check provider configuration
$ terraform providers

# Validate configuration correctness
$ terraform validate
```

### Provider Configuration Best Practices

```hcl
# ✅ Recommended: Explicit provider configuration
terraform {
  required_providers {
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.20"
    }
  }
}

provider "kubernetes" {
  # Explicit connection information
}

# ❌ Avoid: Relying on implicit inheritance or inter-module sharing
```

---

## Troubleshooting

### If you still encounter provider configuration issues:

1. **Check Scope**: Ensure provider configuration and usage are at the same level
2. **Validate Syntax**: Use `terraform validate` to check configuration correctness
3. **Check Permissions**: Verify AWS authentication and EKS access permissions are normal
4. **Clear Cache**: Delete `.terraform` directory and reinitialize
5. **Step-by-step Debugging**: Test simple provider configuration first, then add complex logic

### Debugging Commands

```bash
$ terraform init -upgrade    # Reinitialize and upgrade providers
$ terraform providers        # View provider configuration status
$ terraform plan -var-file=... # Detailed execution plan view
$ TF_LOG=DEBUG terraform plan   # Enable detailed logging
```