# EKS IAM OIDC 架構解析

[English](../en/39_eks_iam_oidc_architecture.md) | [繁體中文](../zh-tw/39_eks_iam_oidc_architecture.md) | [日本語](../ja/39_eks_iam_oidc_architecture.md) | [回到索引](../README.md)

## 概述

在 AWS EKS 環境中，為了讓 Kubernetes 中的 Service Account 能夠安全地使用 AWS 服務，需要透過 IAM OIDC (OpenID Connect) 機制建立身份認證橋梁。本文以 AWS Load Balancer Controller 為例，說明這個架構的運作原理。

## AWS IAM 三層架構

AWS IAM 採用三層架構來管理權限：

1. **Policy**：權限清單（定義可以做什麼）
2. **Role**：身份容器（定義誰可以獲得這些權限）
3. **Trust Relationship**：信任關係（定義誰可以扮演這個角色）

## 實際架構範例

### 模組 2：iam-oidc 模組的職責

**功能**：建立基礎的權限定義和認證機制

```hcl
# 建立 OIDC Identity Provider
resource "aws_iam_openid_connect_provider" "eks" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [local.oidc_thumbprint]
  url             = var.oidc_issuer_url
}

# 建立 IAM Policy
resource "aws_iam_policy" "alb_controller" {
  name        = "${local.prefix}-alb-controller-policy"
  description = "IAM policy for AWS Load Balancer Controller"
  policy      = data.http.alb_controller_policy.response_body
}
```

**重點**：
- ❌ **沒有**建立 IAM Role
- ❌ **沒有**建立 Kubernetes Service Account
- ✅ **只建立** IAM Policy（權限清單）
- ✅ **只建立** OIDC Provider（認證機制）

**Policy 來源**：
- 從 AWS 官方 GitHub 下載標準的 IAM policy JSON
- URL: `https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.9.0/docs/install/iam_policy.json`
- 包含 ALB/NLB 管理所需的所有 AWS API 權限

### 模組 3：lb-ctl 模組的職責

**功能**：建立 IAM Role 並分配權限給特定身份

```hcl
# 建立 IAM Role
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

# 將 Policy 附加到 Role
resource "aws_iam_role_policy_attachment" "load_balancer_controller" {
  policy_arn = var.load_balancer_controller_policy_arn
  role       = aws_iam_role.load_balancer_controller.name
}
```

Service account 的定義則如下：
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
此處定義的 `metadata.name/metadata.namespace` 就是在前面 `assume_role_policy` 區塊裡面的 `StringEquals` 中定義的第一個條件，即特定的 service account。

## OIDC 信任策略詳解

### 雙重驗證機制

在 `assume_role_policy` 的 `Condition` 中，有兩個 `StringEquals` 條件：

1. **sub (Subject) 驗證**：
   ```
   "${OIDC_ISSUER}:sub" = "system:serviceaccount:kube-system:aws-load-balancer-controller"
   ```
   - 驗證特定的 Kubernetes Service Account
   - 確保只有指定的 Service Account 可以 assume 這個 role

2. **aud (Audience) 驗證**：
   ```
   "${OIDC_ISSUER}:aud" = "sts.amazonaws.com"
   ```
   - 驗證 OIDC token 的目標受眾
   - 確保 token 是由 AWS STS 發出，防止 token 被其他服務濫用

### 安全性原則

這兩個條件必須**同時滿足**才能成功 assume role：
- **身份驗證**：確認是正確的 Service Account
- **來源驗證**：確認 token 來源合法

## 完整認證流程

1. **Kubernetes Service Account** (`aws-load-balancer-controller`)
2. **透過 OIDC** 向 AWS STS 請求臨時憑證
3. **STS 驗證** Service Account 身份和 token 來源
4. **Assume IAM Role** 獲得權限
5. **執行 AWS API** 操作（建立/管理 ALB/NLB）

## 模組分工總結

| 模組 | 職責 | 建立的資源 |
|------|------|------------|
| iam-oidc | 定義權限 | IAM Policy + OIDC Provider |
| lb-ctl | 分配權限 | IAM Role + Role Policy Attachment |

**設計優勢**：
- **職責分離**：權限定義與身份管理分開
- **重複使用**：Policy 可以被多個 Role 使用
- **安全性**：透過雙重驗證確保身份安全
- **標準化**：使用 AWS 官方推薦的 Policy 內容

## 相關概念

- **OIDC (OpenID Connect)**：身份認證協議
- **STS (Security Token Service)**：AWS 臨時憑證服務
- **Service Account**：Kubernetes 中 Pod 的身份標識
- **AssumeRoleWithWebIdentity**：透過 Web Identity 獲得 AWS 權限的 API
