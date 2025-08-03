# Terraform Provider 作用域導致的 "no client config" 錯誤

[English](../en/07_terraform_provider_scope_issue.md) | [繁體中文](../zh-tw/07_terraform_provider_scope_issue.md) | [日本語](../ja/07_terraform_provider_scope_issue.md) | [回到索引](../README.md)

---

## 背景
- 實驗日期：2025/08/03
- 難度：🤬
- 描述：嘗試將 Kubernetes provider 配置抽取為共享模組時，出現 "cannot create REST client: no client config" 錯誤。

---

## 問題描述

在設計 Terraform 架構時，為了避免在多個 stack 中重複 Kubernetes provider 配置，嘗試建立共享的 provider 模組，但遇到 Kubernetes resources 無法正確初始化的問題。

### 錯誤訊息範例

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

### 錯誤的架構設計

```hcl
# ❌ 錯誤：嘗試將 provider 配置放在子模組中
module "eks_providers" {
  source = "../../../../modules/shared/eks-providers"
  
  cluster_name           = data.terraform_remote_state.eks_cluster.outputs.cluster_name
  cluster_endpoint       = data.terraform_remote_state.eks_cluster.outputs.cluster_endpoint
  cluster_ca_certificate = data.terraform_remote_state.eks_cluster.outputs.cluster_certificate_authority_data
  aws_region            = "ap-northeast-1"
}

module "eks_storage" {
  source = "../../../../modules/eks-lab/storage"
  # ❌ 這個模組無法使用上面子模組的 provider 配置
  depends_on = [module.eks_providers]
}
```

---

## 問題原因

### 1. Terraform Provider 作用域規則

Terraform 的 provider 配置有嚴格的作用域限制：

- **模組內部隔離**：每個模組都有自己的 provider 作用域
- **向下繼承**：父層級的 provider 配置會自動傳遞給子模組
- **不能橫向共享**：子模組或平行模組的 provider 配置互不影響

### 2. 錯誤理解 `depends_on` 的作用

```
❌ 錯誤認知：depends_on = [module.eks_providers] 能讓配置共享
✅ 實際情況：depends_on 只控制執行順序，不影響 provider 作用域
```

### 3. Provider 初始化流程

```
1. Terraform 讀到 kubernetes_manifest 資源
2. 查找當前作用域的 kubernetes provider 配置
3. 找不到配置 → "cannot create REST client: no client config"
4. 即使存在 depends_on 依賴，也無法存取其他模組的 provider
```

---

## 解決步驟

### 方案一：Stack 层級直接配置 Provider

```hcl
# ✅ 正確：在 stack 層級直接配置 provider
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
  # ✅ 模組自動繼承同層級的 provider 配置
}
```

### 方案二：使用 Terramate 共享模板

#### 步驟 1：建立共享模板

建立檔案 `stacks/dev/shared/terramate-templates/eks-k8s-provider.tm.hcl`：

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

#### 步驟 2：在需要的 Stack 中引入

在 `stack.tm.hcl` 中加入：

```hcl
# 引入共享 provider 模板
import {
  source = "../../shared/terramate-templates/eks-k8s-provider.tm.hcl"
}
```

#### 步驟 3：執行 Terramate 生成

```bash
$ terramate generate
```

---

## 技術原理

### Provider 作用域規則示意圖

```
┌──────────────────────────────────────────────────┐
│                Stack 層級                         │
│  ┌─────────────────────────────────────────────┐ │
│  │  provider "kubernetes" { ... }              │ │
│  └─────────────────────┬───────────────────────┘ │
│                        │ (向下繼承)               │
│  ┌─────────────────────▼───────────────────────┐ │
│  │           module "business_logic"           │ │
│  │  ✅ 可以使用上層的 provider 配置               │ │
│  └─────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│                Stack 層級                         │
│  ┌─────────────────────┐ ┌─────────────────────┐ │
│  │  module "providers" │ │  module "business"  │ │
│  │  內部有 provider {} │ │  ❌ 無法存取左側       │ │
│  └─────────────────────┘ └─────────────────────┘ │
│              │                       ▲           │
│              └───── depends_on ──────┘           │
│                   (只控制執行順序)                 │
└──────────────────────────────────────────────────┘
```

### Kubernetes Provider 初始化流程

```
1. Terraform 掃描 required_providers
   ↓
2. 尋找對應的 provider "kubernetes" 配置
   ↓
3. 使用 host、exec 等參數建立 HTTP client
   ↓ 
4. 呼叫 aws eks get-token 取得 JWT token
   ↓
5. 將 token 加入 Authorization header
   ↓
6. 向 EKS API server 發送請求
```

如果步驟 2 找不到 provider 配置，就會產生 "no client config" 錯誤。

---

## 預防措施

### 1. 理解 Provider 作用域

- ✅ **正確**：在需要使用的層級配置 provider
- ❌ **錯誤**：期望透過子模組或 depends_on 共享 provider

### 2. 選擇適合的共享策略

| 方案 | 優點 | 缺點 | 適用場景 |
|------|------|------|----------|
| Stack 層級配置 | 簡單直接 | 可能重複 | 單一專案 |
| Terramate 模板 | 避免重複，統一管理 | 需要工具支援 | 多環境專案 |
| 全域 provider | 完全共享 | 彈性不足 | 標準化環境 |

### 3. 設計原則

- **Provider 就近原則**：在最接近使用點的層級配置
- **避免模組間依賴**：不要期望模組間能共享 provider
- **文件化配置**：清楚記錄每個 provider 的配置來源

---

## 相關概念

### Terraform Module 系統

```bash
# 查看模組依賴樹
$ terraform graph

# 檢查 provider 配置
$ terraform providers

# 驗證配置正確性
$ terraform validate
```

### Provider 配置最佳實踐

```hcl
# ✅ 推薦：明確的 provider 配置
terraform {
  required_providers {
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.20"
    }
  }
}

provider "kubernetes" {
  # 明確配置連接資訊
}

# ❌ 避免：依賴隱式繼承或模組間共享
```

---

## 故障排除

### 如果仍然遇到 provider 配置問題：

1. **檢查作用域**：確認 provider 配置與使用在同一層級
2. **驗證語法**：使用 `terraform validate` 檢查配置正確性
3. **檢查權限**：確認 AWS 認證和 EKS 存取權限正常
4. **清理快取**：刪除 `.terraform` 目錄重新初始化
5. **逐步除錯**：先測試簡單的 provider 配置，再加入複雜邏輯

### 除錯指令

```bash
$ terraform init -upgrade    # 重新初始化並升級 provider
$ terraform providers        # 查看 provider 配置狀態
$ terraform plan -var-file=... # 詳細查看執行計劃
$ TF_LOG=DEBUG terraform plan   # 開啟詳細日誌
```
