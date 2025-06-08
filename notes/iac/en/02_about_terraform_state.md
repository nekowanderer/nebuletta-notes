# About Terraform State

[English](02_about_terraform_state.md) | [繁體中文](../zh-tw/02_about_terraform_state.md) | [日本語](../ja/02_about_terraform_state.md) | [Back to Index](../README.md)

## Sensitive Information Handling

- ARNs, IDs, and other sensitive information should not appear in `.tf` or `.hcl` files
- These sensitive data are stored in `.tfstate` files
- `.tfstate` files should be added to `.gitignore`

## State File Lifecycle

### During `terraform plan`:
- Terraform reads the remote state
- Compares local code with remote state
- Generates a change plan

### During `terraform apply`:
- Applies the changes
- Updates the remote state
- Local state synchronizes with remote

## Backend Configuration

### What is `backend.hcl`?
- A configuration file that defines where and how to store Terraform state
- Contains static backend settings like bucket name, region, etc.
- Cannot use variables or reference other files
- Should be kept simple and static

### Using `-backend-config`:
- Allows passing backend configuration during initialization
- Can be used to provide different backend settings for different environments
- Example: `terraform init -backend-config=../state-storage/backend.hcl`
- Useful when you need to use the same backend configuration across multiple modules

## Best Practices

1. **Version Control**:
   - `.tfstate` files should not be version controlled
   - Use remote state storage (e.g., S3)
   - Use state locking (e.g., DynamoDB)
   - Keep sensitive information only in state

2. **Why This Approach**:
   - **Security**: Prevents exposure of sensitive information
   - **Collaboration**: Enables safe multi-user infrastructure management
   - **Tracking**: Allows tracking of infrastructure changes
   - **Backup**: State files are safely backed up in S3

## Current Setup

The current configuration of this repo follows these best practices:
- Using S3 for state storage
- Using DynamoDB for state locking
- Sensitive information only exists in state
- Code only contains configuration and logic
