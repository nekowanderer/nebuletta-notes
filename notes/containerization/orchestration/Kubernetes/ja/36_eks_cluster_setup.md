# EKS クラスターセットアップガイド

[English](../en/36_eks_cluster_setup.md) | [繁體中文](../zh-tw/36_eks_cluster_setup.md) | [日本語](../ja/36_eks_cluster_setup.md) | [インデックスに戻る](../README.md)

## 必要な IAM 権限設定

### AWS Managed Policies
この実装は [eks-admin/iam.tf](https://github.com/nekowanderer/nebuletta/blob/main/terraform/modules/compute/ec2/public/eks-admin/iam.tf) の Terraform ソースコードに対応しており、基本的な `AmazonSSMManagedInstanceCore` ポリシーに加えて、以下のポリシーが必要です：
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

### インラインポリシー
さらに、eks admin node が EKS クラスターを正常に作成できるようにするために、以下の `eks-admin-policy` を作成する必要があります：
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

## 必要なアクション一覧

### EKS 関連
- `eks:*` (完全な EKS 権限)
- `eks:DescribeClusterVersions`
- `eks:CreateCluster`
- `eks:DescribeCluster`
- `eks:ListClusters`

### サポートサービス
- `ssm:GetParameter*` (Systems Manager パラメータ)
- `kms:CreateGrant`, `kms:DescribeKey` (KMS 暗号化)
- `logs:PutRetentionPolicy` (CloudWatch Logs)
- `iam:*` (IAM 管理)
- `cloudformation:*` (CloudFormation 管理)

## よく使うコマンド

### IAM 権限の確認
```bash
# 現在のアイデンティティを確認
$ aws sts get-caller-identity

# ロールに添付されているすべてのマネージドポリシーをリスト
$ aws iam list-attached-role-policies --role-name dev-eks-admin-ec2-role

# ロールのすべてのインラインポリシーをリスト
$ aws iam list-role-policies --role-name dev-eks-admin-ec2-role

# 特定のインラインポリシーの内容を確認
$ aws iam get-role-policy --role-name dev-eks-admin-ec2-role --policy-name dev-eks-admin-ec2-eks-admin-policy
```

### EKS クラスター管理
```bash
# EKS クラスターを作成（Fargate を使用）
$ eksctl create cluster --name ${CLUSTER_NAME} --version 1.33 --fargate

# EKS クラスターを作成（EC2 Worker Nodes を使用）
$ eksctl create cluster --name ${CLUSTER_NAME} --node-type t3.micro --nodes 1

# EKS クラスターを削除
$ eksctl delete cluster --region=ap-northeast-1 --name=${CLUSTER_NAME}

# すべての EKS クラスターをリスト
$ eksctl get clusters --region=ap-northeast-1

# kubeconfig を更新
$ aws eks update-kubeconfig --region ap-northeast-1 --name ${CLUSTER_NAME}
```

### CloudFormation 管理
```bash
# 失敗した CloudFormation スタックを削除
$ aws cloudformation delete-stack --stack-name eksctl-${CLUSTER_NAME}-cluster --region ap-northeast-1

# CloudFormation スタックの状態を確認
$ aws cloudformation describe-stacks --stack-name eksctl-${CLUSTER_NAME}-cluster --region ap-northeast-1

# CloudFormation イベントを確認
$ aws cloudformation describe-stack-events --stack-name eksctl-${CLUSTER_NAME}-cluster --region ap-northeast-1
```

### 基本的な Kubernetes 操作
```bash
# クラスターの状態を確認
$ kubectl cluster-info

# ノードを確認
$ kubectl get nodes

# すべての Pod を確認
$ kubectl get pods --all-namespaces

# サービスを確認
$ kubectl get services --all-namespaces
```

## 重要な注意事項
1. **IAM 権限変更後は EC2 インスタンスを再起動する必要があります**
2. **EKS クラスターの作成には 10-15 分かかります**
3. **EKS は高価なサービスです。使用後はクラスターを削除してコストを節約してください**（[料金表](https://aws.amazon.com/eks/pricing/)）
4. **失敗した CloudFormation スタックは手動で削除する必要があります**