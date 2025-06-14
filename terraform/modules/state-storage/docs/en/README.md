# Terraform State Management

[English](README.md) | [繁體中文](../zh-tw/README.md) | [日本語](../ja/README.md) | [Back to Index](../README.md)

This module is used to create and manage remote Terraform state storage. It creates:
- S3 bucket for storing Terraform state files
- DynamoDB table for state locking

## Prerequisites

- AWS CLI installed and configured
- AWS SSO login completed
- Using a profile with sufficient permissions

## Usage

1. Initialize Terraform:
```bash
terraform init
```

2. Run plan:
```bash
terraform plan -var-file="common.tfvars"
```

3. Apply changes:
```bash
terraform apply -var-file="common.tfvars"
```

4. Destroy resources (if needed):
```bash
terraform destroy -var-file="common.tfvars"
```

## Variables

| Variable Name | Description | Example |
|--------------|-------------|---------|
| env | Environment name | dev |
| module_name | Module name | infra |
| aws_region | AWS region | ap-northeast-1 |

## Outputs

| Output Name | Description |
|------------|-------------|
| storage_name | S3 bucket name |
| storage_arn | S3 bucket ARN |
| lock_table_name | DynamoDB table name |
| lock_table_arn | DynamoDB table ARN |

## Notes

1. Resources created by this module are for Terraform state storage only
2. Ensure using the correct AWS profile
3. It's recommended not to set `prevent_destroy` in development environments
4. The tfstate of the remote state storage itself is typically not stored in S3; its purpose is simply to create an empty bucket and a DynamoDB table for locking in AWS
