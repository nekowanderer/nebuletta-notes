# Troubleshooting Documentation

← [Back to Nebuletta Notes](../README.md)

This directory contains real-world troubleshooting experiences and debugging guides in multiple languages:

- [English](en/)
  - [01_Network ACL Impact on Private Subnet Communication - Debugging Experience](en/01_network_acl_private_subnet_troubleshooting.md)
  - [02_AWS Flow Log Could Not Be Deleted - Debugging Experience](en/02_aws_flow_log_could_not_be_deleted.md)
  - [03_Resource Mutation Caused by Terraform State Misconfiguration](en/03_incorrect_shared_state_behavior.md)
  - [04_Networking Options for ECS Tasks under Fargate Launch Type](en/04_ecs_service_target_group_type_setting.md)
  - [05_Dynamic AZ and CIDR Calculation Guide](en/05_calculate_az_and_cidr_dynamically.md)
- [繁體中文](zh-tw/)
  - [01_Network ACL 對 Private Subnet 通信影響的除錯經驗](zh-tw/01_network_acl_private_subnet_troubleshooting.md)
  - [02_AWS Flow Log 無法被正確刪除的除錯經驗](zh-tw/02_aws_flow_log_could_not_be_deleted.md)
  - [03_Terraform State 管理錯誤導致的資源異動問題](zh-tw/03_incorrect_shared_state_behavior.md)
  - [04_Fargate Lunch Type 之下的 ECS Task 的 Networking Options](zh-tw/04_ecs_service_target_group_type_setting.md)
  - [05_動態 AZ 和 CIDR 計算說明](zh-tw/05_calculate_az_and_cidr_dynamically.md)
- [日本語](ja/)
  - [01_Network ACLがPrivate Subnet通信に与える影響のデバッグ経験](ja/01_network_acl_private_subnet_troubleshooting.md)
  - [02_AWS Flow Log が削除できない問題のトラブルシューティング](ja/02_aws_flow_log_could_not_be_deleted.md)
  - [03_Terraform状態の誤設定によるリソース削除の問題](ja/03_incorrect_shared_state_behavior.md)
  - [04_Fargate Launch Type における ECS Task のネットワークオプション](ja/04_ecs_service_target_group_type_setting.md)
  - [05_動的AZとCIDR計算ガイド](ja/05_calculate_az_and_cidr_dynamically.md)

## Structure

- `en/` - English troubleshooting guides
- `zh-tw/` - Traditional Chinese troubleshooting guides
- `ja/` - Japanese troubleshooting guides

## Format

Each troubleshooting guide includes:
- **Background**: Experiment context and difficulty level
- **Symptoms**: What was observed during the issue
- **Debugging Process**: Systematic elimination of possible causes
- **Root Cause**: Technical explanation of the actual problem
- **Solution**: Step-by-step resolution with code examples
- **Prevention**: How to avoid similar issues in the future
- **Key Learnings**: Important takeaways and insights
