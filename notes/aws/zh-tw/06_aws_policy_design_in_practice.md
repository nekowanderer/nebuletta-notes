# AWS Policy 設計實務：S3、SQS 權限控管方式解析

[English](../en/06_aws_policy_design_in_practice.md) | [繁體中文](06_aws_policy_design_in_practice.md) | [日本語](../ja/06_aws_policy_design_in_practice.md) | [回到索引](../README.md)


## 背景
新服務 (AWS account ID: 055667788123) 需要同時利用到 AWS S3 與 SQS (沒有跨帳號存取)，為此需要申請相對應的 IAM policy，範例如下：

#### S3

為 service role 添加以下 IAM policy (identity-based policy):

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

為 S3 bucket 添加以下 policy (resource-based policy):

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

為 service role 添加以下 IAM policy:

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


## 為什麼 S3 需要設定兩邊 (IAM Policy + Bucket Policy)，但 SQS 只需要設定一邊 (IAM Policy)？

#### S3 權限控管的原理

S3 Bucket 本身有自己的權限邏輯（Bucket Policy）
 - S3 預設會檢查 Bucket Policy，有授權才能存取 bucket。
 - 即使 IAM Policy 已經允許，S3 Bucket Policy 沒開放還是會被擋下來。

IAM Role Policy 只代表「這個角色有沒有權限做 S3 相關操作」
 - 如果只有 IAM Policy，S3 bucket 沒授權這個 Principal，還是無法操作。

這是一種雙重授權機制
 - 角色要有權限，bucket 也要允許才行。
 - 任何一邊沒設定都會 Access Denied。

#### SQS 權限控管的原理

SQS 採用「單一授權」
 - SQS 的權限控制全部依賴 IAM Policy。
 - SQS queue 本身預設就是 private，只有被 IAM Policy 授權的 role/user 才能操作 queue。

SQS 有支援 Resource Policy，但 99% 情境都不需要特別設定（多用在 cross-account 時）
 - 一般 queue 不會額外開 policy，預設就只能 IAM 授權才可操作。


## 小結

- S3 = 雙重門鎖（IAM Policy + Bucket Policy）
  - 兩邊都要設定，否則會互相卡住。

- SQS = 單一門鎖（IAM Policy）
  - 只要有 IAM Policy 授權即可，queue 本身不擋。

這也是為什麼在設定 S3 存取權時，會常看到要同時修改「Service IAM Policy」跟「Bucket Policy」兩張票，而 SQS 只要 Service IAM Policy 就夠。

AWS 並沒有限制或要求 S3 這樣的 resource 一定要有雙重授權機制，但這算是一種 best practice:
- 最小權限原則（Principle of Least Privilege）
  - identity-based policy 控制「誰」可以做什麼
  - resource-based policy 控制「這個資源要不要接受這個主體來存取」
  - 這樣可以做「雙重把關」，降低人為疏失或未來變更帶來的風險
- 防止意外開放（例如：日後有新的人/role 被賦予過度權限時，bucket policy 仍會檔下來）
- 跨帳號（Cross-account）需求未來很常見，到時要改更麻煩，不如一開始就統一用 bucket policy 管理

> 要授予存取權限，可以使用身份型政策（identity-based policy）或資源型政策（resource-based policy）其中之一。不過，為了更嚴謹的安全控管，建議兩者都設定：透過身份型政策實現最小權限原則，並用資源型政策控制哪些主體（principal）可以存取該資源。


## 補充

#### 關於 Principal
根據官方的定義：
> Principals are entities in AWS that can perform actions and access resources. A principal can be an AWS account root user, an IAM user, or a role. A principal that represents the identity of an AWS service is a service principal. Use the Principal element in role trust policies to define the principals that you trust to assume the role.

所以：
- Principal 就是誰（哪一個主體）被授權可以存取這個資源。
- 在 resource-based policy（例如 S3 Bucket Policy、SQS Queue Policy）裡，Principal 指的是被授權方。

常見型態：
- AWS 帳號
  - "Principal": { "AWS": "arn:aws:iam::<account-id>:root" }
- IAM Role
  - "Principal": { "AWS": "arn:aws:iam::<account-id>:role/role-name" }
- IAM User
  - "Principal": { "AWS": "arn:aws:iam::<account-id>:user/user-name" }
- AWS Service
  - "Principal": { "Service": "ec2.amazonaws.com" }

#### SQS 預設為 private 的意思

SQS queue 剛建立時，只有同一個 AWS 帳號裡，擁有適當 IAM Policy 的 user/role 可以對它進行操作（像是 send/receive/delete）。
- 換句話說：沒有人能直接對這個 queue 做操作，除非他被賦予了 IAM Policy。

跟「public」或「開放」的對比
- public S3 bucket：代表這個 bucket 任何人都能讀（如果有設定 public-read）。
- SQS queue 沒有所謂 public 狀態，預設就只有自己帳號內被授權的人能操作，外人無法存取。

為什麼會這樣？
- 因為 SQS queue 預設沒有 resource-based policy（queue policy）允許別的帳號或所有人操作。
- 就算知道 queue 的 ARN 或 URL，沒被授權還是會被拒絕（AccessDenied）。

## 參考文獻
- [IAM roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html)
- [Identity-based policies and resource-based policies](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_identity-vs-resource.html)
- [Policy evaluation for requests within a single account](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic_policy-eval-basics.html)
- [AWS services that work with IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_aws-services-that-work-with-iam.html)
- [Configuring an access policy in Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-configure-add-permissions.html)
