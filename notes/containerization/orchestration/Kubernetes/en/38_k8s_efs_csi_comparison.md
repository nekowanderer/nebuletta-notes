# EFS CSI Driver: Manual vs Terraform Implementation Comparison

[English](../en/38_k8s_efs_csi_comparison.md) | [繁體中文](../zh-tw/38_k8s_efs_csi_comparison.md) | [日本語](../ja/38_k8s_efs_csi_comparison.md) | [Back to Index](../README.md)

## Overview

This document compares the manual EFS CSI driver installation method (from [AWS EKS Practical Guide](./35_eks_practice.md)) with a complete Terraform implementation, highlighting the differences and explaining why the manual approach is incomplete.

## Key Point: Manual Approach is Incomplete

The manual installation in the implementation guide only installs **CSI Driver registration, but does not deploy the actual CSI Controller** that performs dynamic provisioning.

## Manual vs Terraform Implementation Comparison

### Manual Approach (from Implementation Guide)

```bash
# Step 1: Manually create EFS file system (AWS Console)
# Step 2: Manually create Security Group (AWS Console)  
# Step 3: Only install CSI Driver registration
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/deploy/kubernetes/base/csidriver.yaml

# Step 4: Manually create static PV
$ kubectl apply -f aws-efs-volume-pv.yaml

# Step 5: Manually create PVC
$ kubectl apply -f simple-volume-pvc.yaml
```

**What actually gets installed:**
- ✅ Only CSI Driver registration
- ❌ No CSI Controller (the actual provisioner)
- ❌ No Service Account for permissions
- ❌ No dynamic provisioning capability

### Terraform Implementation (Complete Solution)

```hcl
# Automated infrastructure creation
resource "aws_efs_file_system" "eks" { ... }           # EFS file system
resource "aws_security_group" "efs" { ... }            # Security group
resource "aws_efs_mount_target" "eks" { ... }          # Mount targets
resource "aws_efs_access_point" "eks" { ... }          # Access points

# Complete CSI infrastructure
resource "kubernetes_service_account" "efs_csi_controller" { ... }    # Permission management
resource "kubernetes_deployment" "efs_csi_controller" { ... }         # Controller
resource "kubernetes_manifest" "efs_csi_driver" { ... }              # Driver registration
resource "kubernetes_storage_class" "efs" { ... }                    # Dynamic provisioning
```

**What it provides:**
- ✅ Complete EFS infrastructure
- ✅ Complete CSI Controller deployment
- ✅ Service Account with appropriate permissions
- ✅ Dynamic provisioning capability
- ✅ Storage Class for applications

## Detailed Component Analysis

### 1. EFS Infrastructure (`efs.tf`)

#### `aws_efs_file_system`
```hcl
resource "aws_efs_file_system" "eks" {
  creation_token   = "${local.prefix}-efs"
  performance_mode = "generalPurpose"
  throughput_mode  = "elastic"           # Auto-scaling throughput
  encrypted        = true                # Encryption at rest
}
```
**Purpose**: Creates the actual EFS file system with optimal configuration for EKS workloads.

#### `aws_security_group`
```hcl
resource "aws_security_group" "efs" {
  ingress {
    from_port   = 2049                   # NFS port
    to_port     = 2049
    protocol    = "tcp"
    cidr_blocks = [data.terraform_remote_state.infra_networking.outputs.vpc_cidr]
  }
}
```
**Purpose**: Controls network access to EFS, allowing only NFS traffic from EKS nodes.

#### `aws_efs_mount_target`
```hcl
resource "aws_efs_mount_target" "eks" {
  count           = length(local.vpc_config_from_remote.private_subnet_ids)
  file_system_id  = aws_efs_file_system.eks.id
  subnet_id       = local.vpc_config_from_remote.private_subnet_ids[count.index]
  security_groups = [aws_security_group.efs.id]
}
```
**Purpose**: Creates mount targets in each availability zone, ensuring high availability.

#### `aws_efs_access_point`
```hcl
resource "aws_efs_access_point" "eks" {
  file_system_id = aws_efs_file_system.eks.id
  posix_user {
    gid = 1000
    uid = 1000
  }
  root_directory {
    path = "/app"
    creation_info {
      owner_gid   = 1000
      owner_uid   = 1000
      permissions = "0755"
    }
  }
}
```
**Purpose**: Provides a secure, controlled entry point to the EFS file system with specific permissions.

### 2. CSI Controller Infrastructure (`efs-csi.tf`)

#### `kubernetes_service_account`
```hcl
resource "kubernetes_service_account" "efs_csi_controller" {
  metadata {
    name      = "${local.prefix}-efs-csi-controller-sa"
    namespace = "kube-system"
    labels = {
      "app.kubernetes.io/name" = "aws-efs-csi-driver"
    }
  }
}
```
**Purpose**: Provides the CSI controller with necessary Kubernetes permissions for volume management.

#### `kubernetes_deployment` (CSI Controller)
```hcl
resource "kubernetes_deployment" "efs_csi_controller" {
  spec {
    replicas = 2 # High availability
    template {
      spec {
        service_account_name = kubernetes_service_account.efs_csi_controller.metadata[0].name
        
        container {
          name  = "${local.prefix}-efs-plugin"
          image = "amazon/aws-efs-csi-driver:v2.0.7"
          # CSI controller that handles volume operations
        }
        
        container {
          name  = "${local.prefix}-csi-provisioner"
          image = "registry.k8s.io/sig-storage/csi-provisioner:v4.0.1"
          # Kubernetes CSI provisioner sidecar
        }
      }
    }
  }
}
```
**Purpose**:
- **EFS Plugin**: Handles AWS EFS-specific operations
- **CSI Provisioner**: Standard Kubernetes CSI sidecar that watches PVC requests and triggers volume creation

#### `kubernetes_manifest` (CSI Driver)
```hcl
resource "kubernetes_manifest" "efs_csi_driver" {
  manifest = {
    apiVersion = "storage.k8s.io/v1"
    kind       = "CSIDriver"
    metadata = {
      name = "efs.csi.aws.com"
    }
    spec = {
      attachRequired = false             # EFS doesn't require attachment
    }
  }
}
```
**Purpose**: Registers the EFS CSI driver with Kubernetes, telling it how to handle EFS volumes.

#### `kubernetes_storage_class`
```hcl
resource "kubernetes_storage_class" "efs" {
  metadata {
    name = "${local.prefix}-efs-sc"
  }
  storage_provisioner = "efs.csi.aws.com"
  parameters = {
    provisioningMode = "efs-ap"          # Use access points
    fileSystemId     = aws_efs_file_system.eks.id
    directoryPerms   = "700"
  }
}
```
**Purpose**: Provides a template for dynamic PV creation. When applications create PVCs referencing this storage class, Kubernetes automatically creates corresponding PVs.

## Why the Manual Approach is Too Simple?

### Missing Components

1. **No CSI Controller**: The manual approach only installs CSI Driver registration but lacks the actual controller that handles provisioning requests.

2. **No Service Account**: Without proper service accounts, CSI components lack necessary permissions.

3. **No Dynamic Provisioning**: Users must manually create static PVs and hard-code EFS file system IDs.

4. **No Storage Class**: Applications cannot dynamically request storage.

### Proper Manual Installation Should Be

- As outlined in the following file:
https://github.com/kubernetes-sigs/aws-efs-csi-driver/blob/master/deploy/kubernetes/overlays/stable/kustomization.yaml

- Or using Helm:
  ```bash
  $ helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
  $ helm install aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver
  ```

## Application Usage Comparison

### Manual Approach (Static Provisioning)
```yaml
# Manually create PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: app-pv
spec:
  storageClassName: sc-001
  capacity:
    storage: 2Gi
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-0a1700a34bf9e8d24  # Hard-coded EFS ID
---
# Then create PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc
spec:
  storageClassName: sc-001
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
```

### Terraform Approach (Dynamic Provisioning)
```yaml
# Application only needs PVC - PV is automatically created
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: prod-app-pvc
  namespace: app-ns
spec:
  storageClassName: dev-eks-storage-efs-sc  # Reference to Terraform-created storage class
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
```

## Advantages of Terraform Implementation

### 1. **Complete Infrastructure as Code**
- All resources are defined and version controlled
- Reproducible across environments
- Automatic dependency management

### 2. **Dynamic Provisioning**
- Applications self-describe storage requirements
- No manual PV creation needed
- Automatic cleanup when PVCs are deleted

### 3. **Proper Security**
- Security groups with minimal required access
- Service accounts following least privilege principle
- Encrypted storage by default

### 4. **High Availability**
- Mount targets across multiple availability zones
- Redundant CSI controller replicas
- Elastic throughput scaling

### 5. **Production Ready**
- Complete logging and monitoring
- Proper resource limits and requests
- Lifecycle management policies

## Conclusion

The manual approach in the implementation guide is oriented towards simple exercises and **is incomplete and unsuitable for production environments**. It only installs CSI Driver registration without the actual provisioning infrastructure, forcing users to manually create static PVs.

The Terraform implementation provides a **complete, production-ready EFS CSI solution** that supports:
- ✅ Complete automation
- ✅ Dynamic provisioning
- ✅ High availability
- ✅ Security best practices
- ✅ Infrastructure as Code principles

**Key Takeaway**: Always verify that manual installation tutorials include all necessary components, not just basic registration steps.