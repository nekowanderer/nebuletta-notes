# Troubleshooting: AWS Flow Log Could Not Be Deleted

[English](02_aws_flow_log_could_not_be_deleted.md) | [繁體中文](../zh-tw/02_aws_flow_log_could_not_be_deleted.md) | [日本語](../ja/02_aws_flow_log_could_not_be_deleted.md) | [Back to Index](../README.md)

---

## Background
- Date: 2025/05/24
- Difficulty: 🤬🤬
- Description: When executing `terraform destroy`, AWS Flow Log and its associated CloudWatch Log Group could not be properly deleted, even though Terraform showed "Destruction complete".

## Observed Issues

- `terraform destroy` executed successfully, showing resources were deleted
- CloudWatch Log Group still visible in AWS Console
- Manual check with AWS CLI confirmed resources still exist:
  ```bash
  aws logs describe-log-groups --region ap-northeast-1 --log-group-name-prefix "/aws/dev-infra-networking"
  ```

## Debugging Process

### Phase 1: Problem Scope Confirmation

Verified the following items:

| Item | Status | Description |
|------|--------|-------------|
| VPC Flow Log | ✅ | Terraform shows deleted |
| CloudWatch Log Group | ❌ | Still exists in AWS Console |
| IAM Role | ✅ | Correctly deleted |
| VPC | ✅ | Correctly deleted |

### Phase 2: Problem Research

1. Checked GitHub Issues
2. Found this is a known AWS Provider issue: [Issue #34996](https://github.com/hashicorp/terraform-provider-aws/issues/34996)
3. Impact:
   - `aws_cloudwatch_log_group` resources
   - Even with `skip_destroy = false`
     - Note: This syntax was deprecated after version `v1.12.0`
   - Also occurs in Terraform Cloud

### Phase 3: Solution Design

Using `null_resource` as a temporary solution:

1. ✅ Use `triggers` to store necessary information
2. ✅ Use `local-exec` provisioner
3. ✅ Execute cleanup script in `destroy` phase
4. ✅ Use AWS CLI to manually delete remaining resources

## Root Cause

### AWS Provider Limitations

1. **Provider Behavior**:
   - Terraform shows resources as deleted
   - But actual AWS resources still exist
   - This is a known Provider issue

2. **Impact Scope**:
   - Affects `aws_cloudwatch_log_group` resources
   - Especially in VPC Flow Log related scenarios
   - Issue also occurs in Terraform Cloud

## Solution

### Using null_resource as a Temporary Solution

```hcl
resource "null_resource" "delete_flow_log" {
  triggers = {
    prefix = local.prefix
  }

  depends_on = [aws_flow_log.vpc_flow_log]

  provisioner "local-exec" {
    when    = destroy
    command = <<-EOT
      # Wait for VPC Flow Log to be deleted
      echo "Waiting for VPC Flow Log to be deleted..."
      sleep 10

      # Check if VPC Flow Log still exists
      FLOW_LOG_ID=$(aws ec2 describe-flow-logs --region ap-northeast-1 --query "FlowLogs[?LogGroupName=='/aws/${self.triggers.prefix}/vpc/flow-logs'].FlowLogId" --output text)
      
      if [ ! -z "$FLOW_LOG_ID" ]; then
        echo "VPC Flow Log still exists, deleting manually..."
        aws ec2 delete-flow-logs --flow-log-ids $FLOW_LOG_ID --region ap-northeast-1
      fi

      # Check if CloudWatch Log Group still exists
      LOG_GROUP_NAME="/aws/${self.triggers.prefix}/vpc/flow-logs"
      if aws logs describe-log-groups --log-group-name-prefix "$LOG_GROUP_NAME" --region ap-northeast-1 --query "logGroups[?logGroupName=='$LOG_GROUP_NAME']" --output text | grep -q "$LOG_GROUP_NAME"; then
        echo "CloudWatch Log Group still exists, deleting manually..."
        aws logs delete-log-group --log-group-name "$LOG_GROUP_NAME" --region ap-northeast-1
      fi
    EOT
  }
}
```

## Key Learnings

### null_resource Characteristics

| Characteristic | Description |
|----------------|-------------|
| Virtual Resource | Does not create actual resources in AWS |
| Lifecycle | Can execute operations in create, update, destroy phases |
| Trigger Conditions | Can save state through triggers |
| Execution Timing | Can control execution timing through when parameter |

### Provisioner Usage

| Type | Purpose | Use Cases |
|------|---------|-----------|
| local-exec | Execute local commands | Resource cleanup, script execution |
| remote-exec | Execute remote commands | Remote host configuration |
| file | Transfer files | Configuration file deployment |

### Solution Pros and Cons

Pros:
- Automated cleanup process
- Independent of Provider's deletion mechanism
- Ensures resources are properly deleted

Cons:
- Requires additional IAM permissions:
  ```json
  {
      "Version": "2012-10-17",
      "Statement": [
          {
              "Effect": "Allow",
              "Action": [
                  "ec2:DeleteFlowLogs",
                  "logs:DeleteLogGroup"
              ],
              "Resource": "*"
          }
      ]
  }
  ```
- Increases Terraform configuration complexity
- Temporary solution

## Preventive Measures

1. **Monitor Provider Updates**:
   - Follow [GitHub Issue #34996](https://github.com/hashicorp/terraform-provider-aws/issues/34996)
   - Wait for official fix

2. **Regular Resource Checks**:
   - Use AWS CLI to check resource status
   - Ensure resources are properly deleted

3. **Log Cleanup Process**:
   - Log cleanup operations
   - Facilitate tracking and debugging

## Conclusion

This debugging experience reminds us that when using Terraform to manage AWS resources, we need to pay special attention to Provider limitations and known issues. Although `null_resource` is not the most elegant solution, it provides an effective temporary solution to ensure resources are properly deleted.

**Remember**: Terraform showing "Destruction complete" doesn't mean resources are actually deleted, especially when dealing with CloudWatch Log Groups! 