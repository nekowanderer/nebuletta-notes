# Terraform State Lock Recovery Guide

[English](../en/06_terraform_state_lock_recovery_guide.md) | [繁體中文](../zh-tw/06_terraform_state_lock_recovery_guide.md) | [日本語](../ja/06_terraform_state_lock_recovery_guide.md) | [Back to Index](../README.md)

---

## Background
- Experiment Date: 2025/08/02
- Difficulty: 🤬
- Description: Error occurred during EKS cluster creation due to AWS SSO session timeout.

---

## Problem Description

When AWS tokens expire during Terraform operations, the following errors may occur:

### Error Message Example

```
│ Error: failed to upload state: operation error S3: PutObject, https response error StatusCode: 400, RequestID: FGTNVZ1CHFZ0N23K, HostID: LJbubun1tq2Ezh0AoRHi0kouNbISTJ2xXWUGtp6Du4imCnwtr8DQZtU1K4ARw2L6UwsF/pPbGg77TulnSrg//xXFtmm432ZZeSQIOdiVPZk=, api error ExpiredToken: The provided token has expired.
│
│
╵
╷
│ Error: Failed to save state
│
│ Error saving state: failed to upload state: operation error S3: PutObject, https response error StatusCode: 400, RequestID: FGTNYRRQXYFHRPNX, HostID:
│ kWV7p03zXxLXGWK2NIMuCaxTvHbYYfY4EezTNeMigRFZsdjyzrHh6qvm4qUSx3rH6b7bwAivwoweHZAiwCVtO5d1pm7/fpmE892YyZQca7Y=, api error ExpiredToken: The provided token has expired.
╵
╷
│ Error: Failed to persist state to backend
│
│ The error shown above has prevented Terraform from writing the updated state to the configured backend. To allow for recovery, the state has been written to the file "errored.tfstate"
│ in the current working directory.
│
│ Running "terraform apply" again at this point will create a forked state, making it harder to recover.
│
│ To retry writing this state, use the following command:
│     terraform state push errored.tfstate
│
╵
╷
│ Error: waiting for EKS Cluster (dev-eks-cluster) create: operation error EKS: DescribeCluster, https response error StatusCode: 403, RequestID: 30e246f7-385f-4274-b154-c6bbae9fd4cf, api error ExpiredTokenException: The security token included in the request is expired
│
│   with module.eks_cluster.aws_eks_cluster.main,
│   on ../../../../modules/eks-lab/cluster/eks_cluster.tf line 13, in resource "aws_eks_cluster" "main":
│   13: resource "aws_eks_cluster" "main" {
│
╵
╷
│ Error: Error releasing the state lock
│
│ Error message: failed to retrieve lock info for lock ID "d27f3f1c-6ce8-c97b-0a3f-e6f77255b429": Unable to retrieve item from DynamoDB table "dev-state-storage-locks": operation error
│ DynamoDB: GetItem, https response error StatusCode: 400, RequestID: 60S67NR6MAIKT5TH73QBL5AHCVVV4KQNSO5AEMVJF66Q9ASUAAJG, api error ExpiredTokenException: The security token included in
│ the request is expired
│
│ Terraform acquires a lock when accessing your state to prevent others
│ running Terraform to potentially modify the state at the same time. An
│ error occurred while releasing this lock. This could mean that the lock
│ did or did not release properly. If the lock didn't release properly,
│ Terraform may not be able to run future commands since it'll appear as if
│ the lock is held.
│
│ In this scenario, please call the "force-unlock" command to unlock the
│ state manually. This is a very dangerous operation since if it is done
│ erroneously it could result in two people modifying state at the same time.
│ Only call this command if you're certain that the unlock above failed and
│ that no one else is holding a lock.
╵
```

---

## Root Causes

1. **AWS Token Expiration**: AWS session token expires during long-running deployments
2. **State Lock Stuck**: Terraform state lock in DynamoDB cannot be properly released
3. **State File Separation**: Terraform writes current state to local `errored.tfstate` file

---

## Resolution Steps

### Step 1: Force Unlock State Lock

Find the lock ID from the error message, for example: `d27f3f1c-6ce8-c97b-0a3f-e6f77255b429`

```bash
# Syntax: terraform force-unlock <LOCK_ID>
$ terraform force-unlock d27f3f1c-6ce8-c97b-0a3f-e6f77255b429
```

The system will ask for confirmation:
```
Do you really want to force-unlock?
  Terraform will remove the lock on the remote state.
  This will allow local Terraform commands to modify this state, even though it
  may still be in use. Only 'yes' will be accepted to confirm.

  Enter a value: yes
```

Type `yes` to confirm.

**⚠️ Warning**: This is a dangerous operation. Only execute when certain no one else is operating on the same state.

### Step 2: Check and Update AWS Credentials

```bash
# Check current AWS authentication status
$ aws sts get-caller-identity
```

If authentication expiration error appears, re-authenticate:
```bash
# AWS SSO login (according to your configuration)
$ aws sso login --profile your-profile

# Or use other authentication methods
$ aws configure
```

### Step 3: Push State File

Push the local `errored.tfstate` back to the remote backend:

```bash
$ terraform state push errored.tfstate
```

After success, you can delete the local error file:
```bash
$ rm errored.tfstate
```

### Step 4: Verify State Recovery

```bash
# Check state status
$ terraform state list

# View plan to ensure no anomalies
$ terraform plan
```

---

## Technical Principles

### Terraform State Lock Mechanism

- **Purpose**: Prevent multiple people from modifying the same infrastructure simultaneously
- **Implementation**: Use DynamoDB table to store lock information
- **Problem**: When operations are abnormally interrupted, locks may not be properly released

### State File Management

- **Remote Backend**: Production environment state stored in S3, locked by DynamoDB
- **Local Backup**: When operations fail, Terraform saves current state to local `errored.tfstate`
- **Recovery**: Use `terraform state push` to push local state back to remote

### AWS Token Expiration Handling

- **Cause**: Long-running operations may exceed token validity period
- **Impact**: Cannot access S3 backend and DynamoDB lock table
- **Solution**: Re-authenticate and continue operations

---

## Prevention Measures

1. **Monitor Token Validity**: Check remaining token time before long operations
2. **Staged Deployment**: Split large infrastructure into multiple smaller modules
3. **Use Long-term Credentials**: Use longer validity authentication when security permits
4. **Regular Lock Checks**: Check for residual state locks before operations

---

## Related Command Reference

```bash
# View current state lock status
$ terraform providers lock

# List all resources in state
$ terraform state list

# View specific resource's state
$ terraform state show <resource_name>

# Import existing resources to state
$ terraform import <resource_type>.<resource_name> <resource_id>

# Remove resources from state (without deleting actual resources)
$ terraform state rm <resource_name>
```

---

## Troubleshooting

If the above steps cannot resolve the issue:

1. **Check AWS Permissions**: Ensure sufficient permissions to access S3 bucket and DynamoDB table
2. **Check Network Connection**: Ensure normal access to AWS services
3. **Check Backend Configuration**: Ensure `backend.tf` configuration is correct
4. **Contact Team**: If in shared environment, confirm no one else is operating