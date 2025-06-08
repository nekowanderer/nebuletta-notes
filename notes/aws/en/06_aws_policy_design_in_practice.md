# AWS Policy Design in Practice: Understanding S3 and SQS Permission Management

[English](06_aws_policy_design_in_practice.md) | [繁體中文](../zh-tw/06_aws_policy_design_in_practice.md) | [日本語](../ja/06_aws_policy_design_in_practice.md) | [Back to Index](../README.md)

## Background
A new service (AWS account ID: 055667788123) needs to access both AWS S3 and SQS (no cross-account access required). The following IAM policies are needed:

#### S3

Add the following IAM policy (identity-based policy) to the service role:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowS3Access",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:GetObjectVersion",
                "s3:ListBucket",
                "s3:PutObject",
                "s3:DeleteObject"
            ],
            "Resource": [
                "arn:aws:s3:::communication-service-attachments",
                "arn:aws:s3:::communication-service-attachments/*"
            ]
        }
    ]
}
```

Add the following policy (resource-based policy) to the S3 bucket:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowS3Access",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::055667788123:role/role_communication-service-ecs"
            },
            "Action": [
                "s3:GetObject",
                "s3:GetObjectVersion",
                "s3:ListBucket",
                "s3:PutObject",
                "s3:DeleteObject"
            ],
            "Resource": [
                "arn:aws:s3:::communication-service-attachments",
                "arn:aws:s3:::communication-service-attachments/*"
            ]
        }
    ]
}
```

#### SQS

Add the following IAM policy to the service role:

```json
{
    "Sid": "SQSAcess",
    "Effect": "Allow",
    "Action": [
        "sqs:SendMessage",
        "sqs:ReceiveMessage",
        "sqs:GetQueueAttributes",
        "sqs:DeleteMessage"
    ],
    "Resource": [
        "arn:aws:sqs:ap-southeast-1:055667788123:communication-service-email-attachment-queue",
        "arn:aws:sqs:ap-southeast-1:055667788123:communication-service-email-attachment-dead-letter-queue"
    ]
}
```

## Why does S3 require both IAM Policy and Bucket Policy, while SQS only needs IAM Policy?

#### S3 Permission Control Principles

S3 Bucket has its own permission logic (Bucket Policy)
- S3 checks Bucket Policy by default, requiring authorization to access the bucket.
- Even if IAM Policy allows access, it will be denied if the S3 Bucket Policy doesn't permit it.

IAM Role Policy only represents "whether this role has permission to perform S3 operations"
- With only IAM Policy, if the S3 bucket doesn't authorize this Principal, operations will still fail.

This is a dual authorization mechanism
- Both the role needs permission and the bucket needs to allow access.
- Access will be denied if either side is not configured.

#### SQS Permission Control Principles

SQS uses "single authorization"
- SQS permission control relies entirely on IAM Policy.
- SQS queues are private by default, only roles/users authorized by IAM Policy can operate on the queue.

SQS supports Resource Policy, but it's rarely needed (mostly used for cross-account scenarios)
- Generally, queues don't need additional policies, as they only allow IAM-authorized operations by default.

## Summary

- S3 = Double Lock (IAM Policy + Bucket Policy)
  - Both sides need to be configured, otherwise they will block each other.

- SQS = Single Lock (IAM Policy)
  - Only IAM Policy authorization is needed, the queue itself doesn't block access.

This is why when setting up S3 access, you often see tickets for both "Service IAM Policy" and "Bucket Policy", while SQS only needs Service IAM Policy.

AWS doesn't restrict or require resources like S3 to have dual authorization mechanisms, but it's considered a best practice:
- Principle of Least Privilege
  - identity-based policy controls "who" can do what
  - resource-based policy controls "whether this resource accepts access from this principal"
  - This provides "double verification", reducing risks from human error or future changes
- Prevents accidental exposure (e.g., if new users/roles are granted excessive permissions later, bucket policy will still block access)
- Cross-account requirements are common in the future, and it's more troublesome to change later, so it's better to manage with bucket policy from the start

> To grant access permissions, you can use either identity-based policy or resource-based policy. However, for stricter security control, it's recommended to set both: implement the principle of least privilege through identity-based policy, and control which principals can access the resource through resource-based policy.

## Additional Notes

#### About Principal
According to the official definition:
> Principals are entities in AWS that can perform actions and access resources. A principal can be an AWS account root user, an IAM user, or a role. A principal that represents the identity of an AWS service is a service principal. Use the Principal element in role trust policies to define the principals that you trust to assume the role.

Therefore:
- Principal refers to who (which entity) is authorized to access this resource.
- In resource-based policies (such as S3 Bucket Policy, SQS Queue Policy), Principal refers to the authorized party.

Common types:
- AWS Account
  - "Principal": { "AWS": "arn:aws:iam::<account-id>:root" }
- IAM Role
  - "Principal": { "AWS": "arn:aws:iam::<account-id>:role/role-name" }
- IAM User
  - "Principal": { "AWS": "arn:aws:iam::<account-id>:user/user-name" }
- AWS Service
  - "Principal": { "Service": "ec2.amazonaws.com" }

#### What "SQS is private by default" means

When an SQS queue is first created, only users/roles within the same AWS account with appropriate IAM Policy can operate on it (such as send/receive/delete).
- In other words: no one can directly operate on this queue unless they are granted IAM Policy.

Comparison with "public" or "open"
- public S3 bucket: means anyone can read this bucket (if public-read is configured).
- SQS queue doesn't have a public state, by default only authorized people within your account can operate it, outsiders cannot access it.

Why is this so?
- Because SQS queue doesn't have resource-based policy (queue policy) allowing other accounts or everyone to operate by default.
- Even if you know the queue's ARN or URL, you'll be denied (AccessDenied) without authorization.

## References
- [IAM roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html)
- [Identity-based policies and resource-based policies](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_identity-vs-resource.html)
- [Policy evaluation for requests within a single account](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic_policy-eval-basics.html)
- [AWS services that work with IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_aws-services-that-work-with-iam.html)
- [Configuring an access policy in Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-configure-add-permissions.html) 