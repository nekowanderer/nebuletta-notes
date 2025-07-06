# EKS Cluster Setup Guide

[English](../en/36_eks_cluster_setup.md) | [繁體中文](../zh-tw/36_eks_cluster_setup.md) | [日本語](../ja/36_eks_cluster_setup.md) | [Back to Index](../README.md)

## Required IAM Permission Configuration

### AWS Managed Policies
The implementation corresponds to the Terraform source code in [eks-admin/iam.tf](https://github.com/nekowanderer/nebuletta/blob/main/terraform/modules/compute/ec2/public/eks-admin/iam.tf), which requires the following policies in addition to the basic `AmazonSSMManagedInstanceCore` policy:
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
Furthermore, to enable the EKS admin node to successfully create EKS clusters, you need to create the following `eks-admin-policy`:
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

## Required Actions List

### EKS Related
- `eks:*` (Full EKS permissions)
- `eks:DescribeClusterVersions`
- `eks:CreateCluster`
- `eks:DescribeCluster`
- `eks:ListClusters`

### Supporting Services
- `ssm:GetParameter*` (Systems Manager parameters)
- `kms:CreateGrant`, `kms:DescribeKey` (KMS encryption)
- `logs:PutRetentionPolicy` (CloudWatch Logs)
- `iam:*` (IAM management)
- `cloudformation:*` (CloudFormation management)

## Common Commands

### Check IAM Permissions
```bash
# Confirm current identity
$ aws sts get-caller-identity

# List all managed policies attached to the role
$ aws iam list-attached-role-policies --role-name dev-eks-admin-ec2-role

# List all inline policies of the role
$ aws iam list-role-policies --role-name dev-eks-admin-ec2-role

# View the content of a specific inline policy
$ aws iam get-role-policy --role-name dev-eks-admin-ec2-role --policy-name dev-eks-admin-ec2-eks-admin-policy
```

### EKS Cluster Management
```bash
# Create EKS cluster (using Fargate)
$ eksctl create cluster --name ${CLUSTER_NAME} --version 1.33 --fargate

# Create EKS cluster (using EC2 Worker Nodes)
$ eksctl create cluster --name ${CLUSTER_NAME} --node-type t3.micro --nodes 1

# Delete EKS cluster
$ eksctl delete cluster --region=ap-northeast-1 --name=${CLUSTER_NAME}

# List all EKS clusters
$ eksctl get clusters --region=ap-northeast-1

# Update kubeconfig
$ aws eks update-kubeconfig --region ap-northeast-1 --name ${CLUSTER_NAME}
```

### CloudFormation Management
```bash
# Delete failed CloudFormation stack
$ aws cloudformation delete-stack --stack-name eksctl-${CLUSTER_NAME}-cluster --region ap-northeast-1

# Check CloudFormation stack status
$ aws cloudformation describe-stacks --stack-name eksctl-${CLUSTER_NAME}-cluster --region ap-northeast-1

# View CloudFormation events
$ aws cloudformation describe-stack-events --stack-name eksctl-${CLUSTER_NAME}-cluster --region ap-northeast-1
```

### Basic Kubernetes Operations
```bash
# Check cluster status
$ kubectl cluster-info

# View nodes
$ kubectl get nodes

# View all pods
$ kubectl get pods --all-namespaces

# View services
$ kubectl get services --all-namespaces
```

## Important Notes
1. **EC2 instances must be restarted after IAM permission changes**
2. **EKS cluster creation takes 10-15 minutes**
3. **EKS is an expensive service, remember to delete clusters to save costs** ([Pricing](https://aws.amazon.com/eks/pricing/))
4. **Failed CloudFormation stacks need to be deleted manually**