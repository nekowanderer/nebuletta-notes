# About AWS KMS (Key Management Service)

[English](./12_aws_kms.md) | [繁體中文](../zh-tw/12_aws_kms.md) | [日本語](../ja/12_aws_kms.md) | [Back to Index](../README.md)

### What is KMS?

AWS KMS is a fully managed encryption key management service used to create and control encryption keys for encrypting your data. KMS integrates with other AWS services, allowing you to easily encrypt data stored in these services.

### Key Features

#### 1. Key Management
- Create, import, rotate, delete, and manage encryption keys
- Support for symmetric and asymmetric keys
- Automatic key backup and version control
- Key alias functionality for easy management

#### 2. Encryption Operations
- Provides encryption and decryption APIs
- Supports multiple encryption algorithms
- Can encrypt data and generate data keys
- Supports cross-region key replication

#### 3. Security
- Keys never leave the KMS service
- Hardware Security Module (HSM) option available
- Compliant with various security standards and regulations
- CloudTrail audit logging support

### Use Cases

#### 1. Data Encryption
- S3 object encryption
- DynamoDB table encryption
- EBS volume encryption
- RDS database encryption
- Cognito user pool encryption

#### 2. Application Encryption
- Application data encryption
- Sensitive information protection
- Cross-service data encryption

#### 3. Compliance Requirements
- Data protection regulations
- Security standard compliance
- Audit trail

### KMS Policies for AWS Resources

#### 1. Why Do We Need KMS Policies?
- Control which AWS services can use KMS keys
- Ensure only authorized services can perform encryption/decryption operations
- Comply with the principle of least privilege

#### 2. Common Service KMS Policy Examples

#### S3 Service
```json
{
  "Sid": "AllowS3UseKey",
  "Effect": "Allow",
  "Principal": {
    "Service": "s3.amazonaws.com"
  },
  "Action": [
    "kms:Encrypt",
    "kms:Decrypt",
    "kms:ReEncrypt*",
    "kms:GenerateDataKey*",
    "kms:DescribeKey"
  ],
  "Resource": "*"
}
```

#### DynamoDB Service
```json
{
  "Sid": "AllowDynamoDBUseKey",
  "Effect": "Allow",
  "Principal": {
    "Service": "dynamodb.amazonaws.com"
  },
  "Action": [
    "kms:Encrypt",
    "kms:Decrypt",
    "kms:ReEncrypt*",
    "kms:GenerateDataKey*",
    "kms:DescribeKey"
  ],
  "Resource": "*"
}
```

#### 3. Policy Configuration Considerations
- Grant only necessary permissions
- Use service-specific principals
- Regularly audit policy settings
- Monitor key usage

### KMS Policies for IAM Roles

#### 1. Why Do We Need IAM Role KMS Policies?
- Control which users/roles can use KMS keys
- Manage encryption/decryption operation permissions
- Implement granular access control

#### 2. Common IAM Role Policy Examples

##### Administrator Role
```json
{
  "Sid": "AllowAccountRoot",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::ACCOUNT_ID:root"
  },
  "Action": [
    "kms:*"  # Full administrative permissions
  ],
  "Resource": "*"
}
```

##### Developer Role
```json
{
  "Sid": "AllowDeveloperAccess",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::ACCOUNT_ID:role/DeveloperRole"
  },
  "Action": [
    "kms:Encrypt",
    "kms:Decrypt",
    "kms:ReEncrypt",
    "kms:GenerateDataKey",
    "kms:DescribeKey"
  ],
  "Resource": "*"
}
```

##### SSO User Role
```hcl
{
  "Sid": "AllowSSOUserAccess",
  "Effect": "Allow",
  "Principal": {
    "AWS": [
      "arn:aws:sts::ACCOUNT_ID:assumed-role/SSO_ROLE/user1",
      "arn:aws:sts::ACCOUNT_ID:assumed-role/SSO_ROLE/user2"
    ]
  },
  "Action": [
    "kms:Encrypt",
    "kms:Decrypt",
    "kms:ReEncrypt",
    "kms:GenerateDataKey",
    "kms:DescribeKey"
  ],
  "Resource": "*"
}
```

#### 3. IAM Role Policy Best Practices

##### Permission Levels
- Administrators: Full permissions (kms:*)
- Developers: Basic operation permissions
- General users: Limited operation permissions

##### Security Considerations
- Follow the principle of least privilege
- Regularly audit permission settings
- Monitor for unusual usage
- Implement permission separation

##### Policy Management
- Use groups to manage permissions
- Regularly rotate keys
- Log all key operations
- Set appropriate alerts

#### 4. Common Use Cases

##### Application Encryption
- Applications need to encrypt sensitive data
- Use IAM roles to access KMS
- Implement encryption/decryption logic

##### Cross-Service Encryption
- Multiple services use the same KMS key
- Control access through IAM roles
- Ensure data consistency

##### Compliance Requirements
- Meet data protection regulations
- Implement access controls
- Provide audit trails

### KMS Key Usage Methods

#### 1. Direct vs. Indirect Usage

##### Direct Usage (Not Recommended)
- Directly access KMS API through IAM roles/users
- Need to manually handle encryption/decryption logic
- Need to manage key lifecycle
- Prone to security risks

##### Indirect Usage (Recommended)
- Automatic encryption/decryption through AWS services
- Examples:
  - S3 automatic object encryption
  - DynamoDB automatic data encryption
  - EBS automatic disk encryption
- More secure and easier to manage

#### 2. Why Indirect Usage is Recommended?

##### Security
- Keys never leave the KMS service
- Reduces human error
- Lowers security risks
- Complies with the principle of least privilege

##### Convenience
- Automatic encryption/decryption handling
- No additional code needed
- Reduces maintenance costs
- Easier to meet compliance requirements

##### Cost Effectiveness
- Reduces API call frequency
- Lowers operational error risks
- Simplifies monitoring and auditing
- Better resource utilization

### Cost Considerations

#### 1. Key Costs
- Customer Managed Keys (CMK): $1/month per key
- AWS Managed Keys: Free

#### 2. API Call Costs
- $0.03 per API call
- Includes encryption, decryption, data key generation, etc.

#### 3. Cost Optimization Recommendations
- Use AWS managed keys when possible
- Consolidate key usage
- Avoid unnecessary key rotation
- Monitor API call frequency

### Best Practices

#### 1. Key Management
- Use key aliases instead of key IDs
- Regularly rotate keys
- Use appropriate IAM permissions
- Enable CloudTrail logging

#### 2. Security
- Follow the principle of least privilege
- Regularly audit key usage
- Monitor for unusual activities
- Use appropriate encryption algorithms

#### 3. Cost Control
- Choose appropriate key types
- Optimize API calls
- Regularly check usage
- Remove unused keys

### Limitations

#### 1. Technical Limitations
- Encryption data size limits per key
- API call limits
- Key quantity limits

#### 2. Cost Limitations
- Key costs
- API call costs
- Data transfer costs

### Summary

AWS KMS provides a secure, reliable, and easy-to-use key management solution. It seamlessly integrates with other AWS services, allowing you to easily implement data encryption requirements. While there are some costs involved, considering the security and convenience it provides, these costs are worthwhile. In practice, you should choose appropriate key types and management strategies based on specific needs, and follow best practices to ensure security and cost-effectiveness. 