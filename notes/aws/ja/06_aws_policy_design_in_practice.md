# AWS ポリシー設計の実践：S3とSQSの権限管理方法の解説

[English](../en/06_aws_policy_design_in_practice.md) | [繁體中文](../zh-tw/06_aws_policy_design_in_practice.md) | [日本語](06_aws_policy_design_in_practice.md) | [インデックスに戻る](../README.md)

## 背景
新しいサービス（AWS アカウント ID: 055667788123）が AWS S3 と SQS の両方にアクセスする必要があります（クロスアカウントアクセスは不要）。以下の IAM ポリシーが必要です：

#### S3

サービスロールに以下の IAM ポリシー（identity-based policy）を追加します：

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

S3 バケットに以下のポリシー（resource-based policy）を追加します：

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

サービスロールに以下の IAM ポリシーを追加します：

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

## なぜ S3 は IAM ポリシーとバケットポリシーの両方が必要なのに、SQS は IAM ポリシーだけで良いのか？

#### S3 の権限管理の仕組み

S3 バケットは独自の権限ロジック（バケットポリシー）を持っています
- S3 はデフォルトでバケットポリシーをチェックし、バケットへのアクセスには認証が必要です。
- IAM ポリシーで許可されていても、S3 バケットポリシーで許可されていない場合はアクセスが拒否されます。

IAM ロールポリシーは「このロールが S3 の操作を行う権限があるかどうか」のみを表します
- IAM ポリシーだけでは、S3 バケットがこのプリンシパルを認証していない場合、操作は失敗します。

これは二重認証メカニズムです
- ロールに権限が必要で、バケットもアクセスを許可する必要があります。
- どちらかが設定されていない場合はアクセスが拒否されます。

#### SQS の権限管理の仕組み

SQS は「単一認証」を使用します
- SQS の権限管理は完全に IAM ポリシーに依存します。
- SQS キューはデフォルトでプライベートであり、IAM ポリシーで認証されたロール/ユーザーのみがキューを操作できます。

SQS はリソースポリシーをサポートしていますが、ほとんど必要ありません（主にクロスアカウントシナリオで使用）
- 一般的に、キューは追加のポリシーを必要とせず、デフォルトでは IAM 認証された操作のみを許可します。

## まとめ

- S3 = 二重ロック（IAM ポリシー + バケットポリシー）
  - 両方の設定が必要で、そうでない場合は互いにブロックされます。

- SQS = 単一ロック（IAM ポリシー）
  - IAM ポリシーの認証のみが必要で、キュー自体はアクセスをブロックしません。

これが、S3 アクセスの設定時に「サービス IAM ポリシー」と「バケットポリシー」の両方のチケットがよく見られるのに対し、SQS はサービス IAM ポリシーだけで十分な理由です。

AWS は S3 のようなリソースに二重認証メカニズムを強制したり要求したりはしませんが、これはベストプラクティスとされています：
- 最小権限の原則
  - identity-based policy は「誰が」何をできるかを制御
  - resource-based policy は「このリソースがこのプリンシパルからのアクセスを受け入れるかどうか」を制御
  - これにより「二重確認」が可能になり、人的ミスや将来の変更によるリスクを低減
- 意図しない公開を防止（例：後で新しいユーザー/ロールに過剰な権限が付与された場合でも、バケットポリシーがブロック）
- クロスアカウント要件は将来一般的になるため、後で変更するよりも最初からバケットポリシーで管理する方が良い

> アクセス権限を付与するには、identity-based policy または resource-based policy のいずれかを使用できます。ただし、より厳格なセキュリティ管理のために、両方を設定することをお勧めします：identity-based policy で最小権限の原則を実装し、resource-based policy でどのプリンシパルがリソースにアクセスできるかを制御します。

## 補足

#### プリンシパルについて
公式の定義によると：
> プリンシパルは、AWS でアクションを実行し、リソースにアクセスできるエンティティです。プリンシパルは、AWS アカウントのルートユーザー、IAM ユーザー、またはロールである可能性があります。AWS サービスの ID を表すプリンシパルはサービスプリンシパルです。ロール信頼ポリシーの Principal 要素を使用して、ロールを引き受けることを信頼するプリンシパルを定義します。

したがって：
- プリンシパルは、このリソースへのアクセスが認証されている誰（どのエンティティ）かを指します。
- リソースベースのポリシー（S3 バケットポリシー、SQS キューポリシーなど）では、プリンシパルは認証された側を指します。

一般的なタイプ：
- AWS アカウント
  - "Principal": { "AWS": "arn:aws:iam::<account-id>:root" }
- IAM ロール
  - "Principal": { "AWS": "arn:aws:iam::<account-id>:role/role-name" }
- IAM ユーザー
  - "Principal": { "AWS": "arn:aws:iam::<account-id>:user/user-name" }
- AWS サービス
  - "Principal": { "Service": "ec2.amazonaws.com" }

#### 「SQS がデフォルトでプライベート」の意味

SQS キューが最初に作成されたとき、適切な IAM ポリシーを持つ同じ AWS アカウント内のユーザー/ロールのみが操作できます（送信/受信/削除など）。
- つまり：IAM ポリシーが付与されていない限り、誰もこのキューを直接操作することはできません。

「パブリック」または「オープン」との比較
- パブリック S3 バケット：このバケットは誰でも読み取り可能（public-read が設定されている場合）。
- SQS キューにはパブリック状態がなく、デフォルトではアカウント内の認証された人だけが操作でき、外部の人はアクセスできません。

なぜそうなるのか？
- SQS キューはデフォルトで他のアカウントや全員が操作できるリソースベースのポリシー（キューポリシー）を持っていないためです。
- キューの ARN や URL を知っていても、認証がないと拒否（AccessDenied）されます。

## 参考文献
- [IAM roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html)
- [Identity-based policies and resource-based policies](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_identity-vs-resource.html)
- [Policy evaluation for requests within a single account](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic_policy-eval-basics.html)
- [AWS services that work with IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_aws-services-that-work-with-iam.html)
- [Configuring an access policy in Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-configure-add-permissions.html) 