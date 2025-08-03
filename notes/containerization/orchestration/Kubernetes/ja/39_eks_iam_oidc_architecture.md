# EKS IAM OIDC アーキテクチャ解析

[English](../en/39_eks_iam_oidc_architecture.md) | [繁體中文](../zh-tw/39_eks_iam_oidc_architecture.md) | [日本語](../ja/39_eks_iam_oidc_architecture.md) | [インデックスに戻る](../README.md)

## 概要

AWS EKS環境において、KubernetesのService AccountがAWSサービスを安全に使用できるようにするため、IAM OIDC（OpenID Connect）メカニズムを通じて身元認証ブリッジを構築する必要があります。本文書では、AWS Load Balancer Controllerを例に、このアーキテクチャの動作原理を説明します。

## AWS IAM 三層アーキテクチャ

AWS IAMは権限管理に三層アーキテクチャを採用しています：

1. **Policy**：権限リスト（何ができるかを定義）
2. **Role**：アイデンティティコンテナ（誰がこれらの権限を取得できるかを定義）
3. **Trust Relationship**：信頼関係（誰がこのロールを引き受けることができるかを定義）

## 実際のアーキテクチャ例

### モジュール2：iam-oidcモジュールの責務

**機能**：基本的な権限定義と認証メカニズムの構築

```hcl
# OIDC Identity Providerの作成
resource "aws_iam_openid_connect_provider" "eks" {
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [local.oidc_thumbprint]
  url             = var.oidc_issuer_url
}

# IAM Policyの作成
resource "aws_iam_policy" "alb_controller" {
  name        = "${local.prefix}-alb-controller-policy"
  description = "IAM policy for AWS Load Balancer Controller"
  policy      = data.http.alb_controller_policy.response_body
}
```

**重要なポイント**：
- ❌ IAM Roleを**作成しない**
- ❌ Kubernetes Service Accountを**作成しない**
- ✅ IAM Policy（権限リスト）**のみを作成**
- ✅ OIDC Provider（認証メカニズム）**のみを作成**

**Policyのソース**：
- AWS公式GitHubから標準のIAM policy JSONをダウンロード
- URL: `https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.9.0/docs/install/iam_policy.json`
- ALB/NLB管理に必要なすべてのAWS API権限を含む

### モジュール3：lb-ctlモジュールの責務

**機能**：IAM Roleを作成し、特定のアイデンティティに権限を割り当て

```hcl
# IAM Roleの作成
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

# PolicyをRoleにアタッチ
resource "aws_iam_role_policy_attachment" "load_balancer_controller" {
  policy_arn = var.load_balancer_controller_policy_arn
  role       = aws_iam_role.load_balancer_controller.name
}
```

Service Accountの定義は以下の通りです：
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
ここで定義される`metadata.name/metadata.namespace`は、前述の`assume_role_policy`ブロック内の`StringEquals`で定義された第一の条件、つまり特定のservice accountです。

## OIDC信頼ポリシー詳細解説

### 二重認証メカニズム

`assume_role_policy`の`Condition`内には、2つの`StringEquals`条件があります：

1. **sub（Subject）検証**：
   ```
   "${OIDC_ISSUER}:sub" = "system:serviceaccount:kube-system:aws-load-balancer-controller"
   ```
   - 特定のKubernetes Service Accountを検証
   - 指定されたService Accountのみがこのロールをassumeできることを保証

2. **aud（Audience）検証**：
   ```
   "${OIDC_ISSUER}:aud" = "sts.amazonaws.com"
   ```
   - OIDCトークンのターゲット受信者を検証
   - トークンがAWS STSによって発行されたことを確認し、他のサービスによるトークンの悪用を防止

### セキュリティ原則

この2つの条件は**同時に満たされる**必要があり、そうして初めてロールのassumeが成功します：
- **身元認証**：正しいService Accountであることを確認
- **ソース検証**：トークンソースが合法であることを確認

## 完全認証フロー

1. **Kubernetes Service Account**（`aws-load-balancer-controller`）
2. **OIDCを通じて**AWS STSに一時認証情報を要求
3. **STS検証**：Service Accountの身元とトークンソースを検証
4. **IAM Roleを引き受け**権限を取得
5. **AWS API操作を実行**（ALB/NLBの作成/管理）

## モジュール分担まとめ

| モジュール | 責務 | 作成されるリソース |
|-----------|------|-------------------|
| iam-oidc | 権限定義 | IAM Policy + OIDC Provider |
| lb-ctl | 権限割り当て | IAM Role + Role Policy Attachment |

**設計の利点**：
- **責務分離**：権限定義と身元管理が分離されている
- **再利用性**：Policyは複数のRoleで使用可能
- **セキュリティ**：二重認証により身元の安全性を確保
- **標準化**：AWS公式推奨のPolicyコンテンツを使用

## 関連概念

- **OIDC（OpenID Connect）**：身元認証プロトコル
- **STS（Security Token Service）**：AWS一時認証情報サービス
- **Service Account**：KubernetesにおけるPodの身元識別子
- **AssumeRoleWithWebIdentity**：Web Identityを通じてAWS権限を取得するAPI