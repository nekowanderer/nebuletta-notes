# Documentation

← [Back to Nebuletta Notes](../README.md)

This directory contains documentation about Terraform in multiple languages:

- [English](en/)
  - [01. Basic Terraform Commands](en/01_basic_terraform_commands.md)
  - [02. About Terraform State](en/02_about_terraform_state.md)
  - [03. Network ACL (Stateless) vs Security Group (Stateful) Deep Analysis](en/03_network_acl_stateless_vs_security_group_stateful.md)
  - [04. AWS Route Table Design Discussion](en/04_aws_route_table_design.md)
  - [05. ECS, Fargate, Lambda, EKS Comparison and Sidecar Design Explanation](en/05_aws_ecs_fargate_comparison.md)
  - [06. AWS Policy Design in Practice: Understanding S3 and SQS Permission Management](en/06_aws_policy_design_in_practice.md)
  - [07. Route 53 vs Global Accelerator: Properties and Differences](en/07_route53_vs_global_accelerator.md)
  - [08. Practical Guide to Email Attachment Base64 Encoding and Size Calculation in AWS SES](en/08_aws-ses-attachment-base64-sizing.md)
- [繁體中文](zh-tw/)
  - [01. Terraform 基本指令](zh-tw/01_basic_terraform_commands.md)
  - [02. 關於 Terraform 狀態](zh-tw/02_about_terraform_state.md)
  - [03. Network ACL (Stateless) vs Security Group (Stateful) 深度解析](zh-tw/03_network_acl_stateless_vs_security_group_stateful.md)
  - [04. AWS Route Table 設計討論](zh-tw/04_aws_route_table_design.md)
  - [05. ECS、Fargate、Lambda、EKS 比較與 Sidecar 設計說明](zh-tw/05_aws_ecs_fargate_comparison.md)
  - [06. AWS Policy 設計實務：S3、SQS 權限控管方式解析](zh-tw/06_aws_policy_design_in_practice.md)
  - [07. Route 53 與 Global Accelerator 的性質與差異比較](zh-tw/07_route53_vs_global_accelerator.md)
  - [08. 從 AWS SES 實務談 Email 附件 Base64 編碼與容量計算](zh-tw/08_aws-ses-attachment-base64-sizing.md)
- [日本語](ja/)
  - [01. Terraform 基本コマンド](ja/01_basic_terraform_commands.md)
  - [02. Terraform 状態について](ja/02_about_terraform_state.md)
  - [03. Network ACL（ステートレス）vs Security Group（ステートフル）深掘り解析](ja/03_network_acl_stateless_vs_security_group_stateful.md)
  - [04. AWS ルートテーブル設計の議論](ja/04_aws_route_table_design.md)
  - [05. ECS、Fargate、Lambda、EKS の比較と Sidecar 設計の説明](ja/05_aws_ecs_fargate_comparison.md)
  - [06. AWS ポリシー設計の実践：S3とSQSの権限管理方法の解説](ja/06_aws_policy_design_in_practice.md)
  - [07. Route 53 と Global Accelerator の性質と違いの比較](ja/07_route53_vs_global_accelerator.md)
  - [08. AWS SES 実務から見るメール添付ファイルのBase64エンコーディングと容量計算](ja/08_aws-ses-attachment-base64-sizing.md)

## Structure

- `en/` - English documentation
- `zh-tw/` - Traditional Chinese documentation
- `ja/` - Japanese documentation 

## Topics Covered

### Basic Terraform Knowledge
- Essential Terraform commands and workflows
- Understanding Terraform state management
- Best practices for infrastructure as code

### AWS Networking Concepts
- Network ACL vs Security Group comparison
- Stateless vs Stateful networking principles
- Ephemeral ports and bidirectional communication rules 