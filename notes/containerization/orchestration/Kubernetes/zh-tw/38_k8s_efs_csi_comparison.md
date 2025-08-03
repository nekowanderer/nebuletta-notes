# EFS CSI Driver：手動 vs Terraform 實作對比

[English](../en/38_k8s_efs_csi_comparison.md) | [繁體中文](../zh-tw/38_k8s_efs_csi_comparison.md) | [日本語](../ja/38_k8s_efs_csi_comparison.md) | [回到索引](../README.md)

## 概述

本文比較了手動 EFS CSI driver 安裝方式（來自 [AWS EKS 實作指南](./35_eks_practice.md)）與完整的 Terraform 實作，重點說明兩者差異並解釋為什麼手動方式是不完整的。

## 重點：手動方式是不完整的

實作指南中的手動安裝只安裝了 **CSI Driver 註冊，但沒有執行動態佈建的實際 **CSI Controller**。

## 手動 vs Terraform 實作對比

### 手動方式（來自實作指南）

```bash
# 步驟 1: 手動建立 EFS 檔案系統（AWS Console）
# 步驟 2: 手動建立 Security Group（AWS Console）  
# 步驟 3: 只安裝 CSI Driver 註冊
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/deploy/kubernetes/base/csidriver.yaml

# 步驟 4: 手動建立靜態 PV
kubectl apply -f aws-efs-volume-pv.yaml

# 步驟 5: 手動建立 PVC
kubectl apply -f simple-volume-pvc.yaml
```

**實際安裝了什麼：**
- ✅ 只有 CSI Driver 註冊
- ❌ 沒有 CSI Controller（實際的佈建器）
- ❌ 沒有權限用的 Service Account
- ❌ 沒有動態佈建能力

### Terraform 實作（完整解決方案）

```hcl
# 自動化基礎設施建立
resource "aws_efs_file_system" "eks" { ... }           # EFS 檔案系統
resource "aws_security_group" "efs" { ... }            # 安全群組
resource "aws_efs_mount_target" "eks" { ... }          # 掛載目標
resource "aws_efs_access_point" "eks" { ... }          # 存取點

# 完整的 CSI 基礎設施
resource "kubernetes_service_account" "efs_csi_controller" { ... }    # 權限管理
resource "kubernetes_deployment" "efs_csi_controller" { ... }         # Controller
resource "kubernetes_manifest" "efs_csi_driver" { ... }              # Driver 註冊
resource "kubernetes_storage_class" "efs" { ... }                    # 動態佈建
```

**提供了什麼：**
- ✅ 完整的 EFS 基礎設施
- ✅ 完整的 CSI Controller 部署
- ✅ 具有適當權限的 Service Account
- ✅ 動態佈建能力
- ✅ 應用程式用的 Storage Class

## 詳細元件分析

### 1. EFS 基礎設施 (`efs.tf`)

#### `aws_efs_file_system`
```hcl
resource "aws_efs_file_system" "eks" {
  creation_token   = "${local.prefix}-efs"
  performance_mode = "generalPurpose"
  throughput_mode  = "elastic"           # Auto-scaling throughput
  encrypted        = true                # Encryption at rest
}
```
**目的**：建立實際的 EFS 檔案系統，為 EKS 工作負載提供最佳設定。

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
**目的**：控制對 EFS 的網路存取，僅允許來自 EKS 節點的 NFS 流量。

#### `aws_efs_mount_target`
```hcl
resource "aws_efs_mount_target" "eks" {
  count           = length(local.vpc_config_from_remote.private_subnet_ids)
  file_system_id  = aws_efs_file_system.eks.id
  subnet_id       = local.vpc_config_from_remote.private_subnet_ids[count.index]
  security_groups = [aws_security_group.efs.id]
}
```
**目的**：在每個可用區域建立掛載目標，確保高可用性。

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
**目的**：為 EFS 檔案系統提供安全、可控制的進入點，具有特定權限。

### 2. CSI Controller 基礎設施 (`efs-csi.tf`)

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
**目的**：為 CSI controller 提供管理卷冊所需的 Kubernetes 權限。

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
**目的**：
- **EFS Plugin**：處理 AWS EFS 特有的操作
- **CSI Provisioner**：標準的 Kubernetes CSI sidecar，監視 PVC 請求並觸發卷冊建立

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
**目的**：向 Kubernetes 註冊 EFS CSI driver，告訴它如何處理 EFS 卷冊。

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
**目的**：為動態 PV 建立提供範本。當應用程式建立引用此 storage class 的 PVC 時，Kubernetes 會自動建立對應的 PV。

## 為什麼手動方式太陽春？

### 缺少的東西

1. **沒有 CSI Controller**：手動方式只安裝了 CSI Driver 註冊，但沒有實際處理佈建請求的 controller。

2. **沒有 Service Account**：沒有適當的 service account，CSI 元件缺乏必要的權限。

3. **沒有動態佈建**：用戶必須手動建立靜態 PV，並硬編碼 EFS 檔案系統 ID。

4. **沒有 Storage Class**：應用程式無法動態請求儲存。

### 較為正規的手動安裝應該是

- 如以下檔案內容：
https://github.com/kubernetes-sigs/aws-efs-csi-driver/blob/master/deploy/kubernetes/overlays/stable/kustomization.yaml

- 或使用 Helm：
  ```bash
  helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
  helm install aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver
  ```

## 應用程式使用方式對比

### 手動方式（靜態佈建）
```yaml
# 手動建立 PV
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
    volumeHandle: fs-0a1700a34bf9e8d24  # 寫死的 EFS ID
---
# 然後建立 PVC
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

### Terraform 方式（動態佈建）
```yaml
# 應用程式只需要 PVC - PV 會自動建立
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: prod-app-pvc
  namespace: app-ns
spec:
  storageClassName: dev-eks-storage-efs-sc  # 引用 Terraform 建立的 storage class
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
```

## Terraform 實作的優勢

### 1. **完整的基礎設施即程式碼**
- 所有資源都有定義且版本控制
- 跨環境可重現
- 自動依賴管理

### 2. **動態佈建**
- 應用程式自我描述儲存需求
- 無需手動建立 PV
- PVC 刪除時自動清理

### 3. **適當的安全性**
- 安全群組具有最小必要存取權限
- Service account 遵循最小權限原則
- 預設加密儲存

### 4. **高可用性**
- 多個可用區域的掛載目標
- 冗餘的 CSI controller 副本
- 彈性吞吐量擴展

### 5. **生產就緒**
- 完整的日誌記錄和監控
- 適當的資源限制和請求
- 生命週期管理政策

## 結論

實作指南中的手動方式是以簡單練習為導向，**不完整且不適合生產環境**。它只安裝了 CSI Driver 註冊，而沒有實際的佈建基礎設施，強迫使用者手動建立靜態 PV。

Terraform 實作提供了**完整、生產就緒的 EFS CSI 解決方案**，支援：
- ✅ 完整自動化
- ✅ 動態佈建
- ✅ 高可用性
- ✅ 安全性最佳實踐
- ✅ 基礎設施即程式碼原則

**關鍵要點**：始終驗證手動安裝教學包含所有必要元件，而不只是基本的註冊步驟。
