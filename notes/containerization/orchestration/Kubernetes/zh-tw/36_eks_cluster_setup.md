# EKS 建立設定指南

[English](../en/36_eks_cluster_setup.md) | [繁體中文](../zh-tw/36_eks_cluster_setup.md) | [日本語](../ja/36_eks_cluster_setup.md) | [回到索引](../README.md)

## 必要的 IAM 權限設定

### AWS Managed Policies
此處對應到的實作是 [eks-admin/iam.tf](https://github.com/nekowanderer/nebuletta/blob/main/terraform/modules/compute/ec2/public/eks-admin/iam.tf) 的 Terraform 原始碼，其中除了最基本的 `AmazonSSMManagedInstanceCore` policy 之外，還需要以下 policy：
```hcl
resource "aws_iam_role_policy_attachment" "eks_worker_node_policy" {
  role       = aws_iam_role.ec2_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
}

resource "aws_iam_role_policy_attachment" "eks_cni_policy" {
  role       = aws_iam_role.ec2_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
}

resource "aws_iam_role_policy_attachment" "ec2_container_registry_read_only" {
  role       = aws_iam_role.ec2_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
}

resource "aws_iam_role_policy_attachment" "iam_full_access" {
  role       = aws_iam_role.ec2_role.name
  policy_arn = "arn:aws:iam::aws:policy/IAMFullAccess"
}

resource "aws_iam_role_policy_attachment" "cloudformation_full_access" {
  role       = aws_iam_role.ec2_role.name
  policy_arn = "arn:aws:iam::aws:policy/AWSCloudFormationFullAccess"
}
```

### Inline Policy
再來，要讓 eks admin node 可以順利建立 EKS cluster，還需要建立以下 `eks-admin-policy`：
```hcl
resource "aws_iam_role_policy" "eks_admin_policy" {
  name = "${local.prefix}-eks-admin-policy"
  role = aws_iam_role.ec2_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "eks:*",
          "ssm:GetParameter",
          "ssm:GetParameters",
          "ssm:GetParametersByPath",
          "kms:CreateGrant",
          "kms:DescribeKey",
          "logs:PutRetentionPolicy"
        ]
        Resource = "*"
      }
    ]
  })
}
```

## 必要的 Actions 清單

### EKS 相關
- `eks:*` (完整 EKS 權限)
- `eks:DescribeClusterVersions`
- `eks:CreateCluster`
- `eks:DescribeCluster`
- `eks:ListClusters`

### 支援服務
- `ssm:GetParameter*` (Systems Manager 參數)
- `kms:CreateGrant`, `kms:DescribeKey` (KMS 加密)
- `logs:PutRetentionPolicy` (CloudWatch Logs)
- `iam:*` (IAM 管理)
- `cloudformation:*` (CloudFormation 管理)

## 常用指令

### 檢查 IAM 權限
```bash
# 確認當前身份
$ aws sts get-caller-identity

# 列出角色附加的所有 managed policies
$ aws iam list-attached-role-policies --role-name dev-eks-admin-ec2-role

# 列出角色的所有 inline policies
$ aws iam list-role-policies --role-name dev-eks-admin-ec2-role

# 查看特定 inline policy 的內容
$ aws iam get-role-policy --role-name dev-eks-admin-ec2-role --policy-name dev-eks-admin-ec2-eks-admin-policy
```

### EKS 叢集管理
```bash
# 建立 EKS 叢集 (使用 Fargate)
$ eksctl create cluster --name ${CLUSTER_NAME} --version 1.33 --fargate

# 建立 EKS 叢集 (使用 EC2 Worker Nodes)
$ eksctl create cluster --name ${CLUSTER_NAME} --node-type t3.micro --nodes 1

# 刪除 EKS 叢集
$ eksctl delete cluster --region=ap-northeast-1 --name=${CLUSTER_NAME}

# 列出所有 EKS 叢集
$ eksctl get clusters --region=ap-northeast-1

# 更新 kubeconfig
$ aws eks update-kubeconfig --region ap-northeast-1 --name ${CLUSTER_NAME}
```

### CloudFormation 管理
```bash
# 刪除失敗的 CloudFormation stack
$ aws cloudformation delete-stack --stack-name eksctl-${CLUSTER_NAME}-cluster --region ap-northeast-1

# 檢查 CloudFormation stack 狀態
$ aws cloudformation describe-stacks --stack-name eksctl-${CLUSTER_NAME}-cluster --region ap-northeast-1

# 查看 CloudFormation 事件
$ aws cloudformation describe-stack-events --stack-name eksctl-${CLUSTER_NAME}-cluster --region ap-northeast-1
```

### Kubernetes 基本操作
```bash
# 檢查叢集狀態
$ kubectl cluster-info

# 查看節點
$ kubectl get nodes

# 查看所有 pods
$ kubectl get pods --all-namespaces

# 查看服務
$ kubectl get services --all-namespaces
```

## 重要提醒
1. **IAM 權限變更後必須重啟 EC2 實例**
2. **EKS 叢集建立需要 10-15 分鐘**
3. **EKS 是很昂貴的服務，用完記得刪除叢集以節省成本**（[收費基準](https://aws.amazon.com/eks/pricing/)）
4. **失敗的 CloudFormation stack 需要手動刪除**
