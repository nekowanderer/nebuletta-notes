# AWS SES クイックノート

[English](../en/13_aws_ses_quick_note.md) | [繁體中文](../zh-tw/13_aws_ses_quick_note.md) | [日本語](ja/13_aws_ses_quick_note.md) | [インデックスに戻る](../README.md)

## テンプレートの作成
JSON形式のテンプレートを作成：
```bash
$ vi ses_template.json
```

```json
{
    "TemplateName": "ses_template",
    "TemplateContent": {
        "Subject": "Hi, {{name}}",
        "Text": "Hi {{name}},\r\nYour favorite animal is {{favoriteanimal}}.",
        "Html": "<h1>Hi {{name}},</h1><p>Your favorite animal is {{favoriteanimal}}.</p>"
    }
}
```

AWS CLIでアップロード：
```bash
$ aws sesv2 create-email-template --cli-input-json file://ses_template.json
```

アップロードが成功したかチェック：
```bash
$ aws sesv2 list-email-templates | jq
{
  "TemplatesMetadata": [
    {
      "TemplateName": "ses_template",
      "CreatedTimestamp": "2025-06-28T12:41:17.945000+08:00"
    }
  ]
}
```

コマンドでテンプレート内容を確認：
```bash
$ aws sesv2 get-email-template --template-name ses_template | jq
{
  "TemplateName": "ses_template",
  "TemplateContent": {
    "Subject": "Hi, {{name}}",
    "Text": "Hi {{name}},\r\nYour favorite animal is {{favoriteanimal}}.",
    "Html": "<h1>Hi {{name}},</h1><p>Your favorite animal is {{favoriteanimal}}.</p>"
  }
}
```

重複するテンプレートの作成はブロックされます：
```bash
$ aws sesv2 create-email-template --cli-input-json file://ses_template.json
An error occurred (AlreadyExistsException) when calling the CreateEmailTemplate operation: Template ses_template already exists for account id 362395300803
```

テンプレートの更新：
```bash
$ aws sesv2 update-email-template --cli-input-json file://ses_template.json
```

## 単一宛先へのメール送信

設定セットの作成
- AWSコンソールでSES -> Configuration -> Configuration setsに直接移動してテスト用のものを作成、この例では`test-configuration-set`を作成

送信者と受信者のidentityの確認
- SES -> Configuration -> Identitiesで、AWS SESプランとルールに従ってidentity（ドメイン/メールアドレス）を作成

test_email.jsonの作成：

```bash
$ vi test_email.json
```

```json
{
    "FromEmailAddress": "Carl Lu <clu@wandrers.one>",
    "Destination": {
        "ToAddresses": [
            "xxx@gmail.com", "xxx@gmail.com"
        ]
    },
    "Content": {
        "Template": {
            "TemplateName": "ses_template",
            "TemplateData": "{ \"name\":\"Ashelee\", \"favoriteanimal\": \"red panda\" }"
        }
    },
    "ConfigurationSetName": "test-configuration-set"
}
```
- TemplateName: 前のステップで作成したものと一致する必要があります
- TemplateData: 必要なパラメータを含める必要があります。パラメータが間違っている場合、メールが配信されない可能性があり、AWS CLIは明示的にエラーを報告しません（SNSトピックが設定されている場合を除く、これは複雑です）。通常、アプリケーション内でパラメータを先に検証する方が良く、エラーハンドリングも比較的制御しやすくなります

メールの送信：
```bash
aws sesv2 send-email --cli-input-json file://test_email.json | jq
{
  "MessageId": "01060197b4eb2897-6d209e61-ca60-4605-8dc7-474f3ad1eb18-000000"
}
```

## 複数宛先へのメール送信

バッチ送信用のtest_bulk_email.jsonを作成：

```bash
$ vi test_bulk_email.json
```

```json
{
    "FromEmailAddress": "Carl Lu <clu@wandrers.one>",
    "DefaultContent": {
        "Template": {
            "TemplateName": "ses_template",
            "TemplateData": "{ \"name\":\"Ashelee\", \"favoriteanimal\":\"red panda\" }"
        }
    },
    "BulkEmailEntries": [
        {
            "Destination": {
                "ToAddresses": [
                    "xxx@gmail.com"
                ]
            },
            "ReplacementEmailContent": {
                "ReplacementTemplate": {
                    "ReplacementTemplateData": "{ \"name\":\"Nana\", \"favoriteanimal\":\"Fat Lu\" }"
                }
            }
        },
        {
            "Destination": {
                "ToAddresses": [
                    "xxx@gmail.com"
                ]
            },
            "ReplacementEmailContent": {
                "ReplacementTemplate": {
                    "ReplacementTemplateData": "{}"
                }
            }
        }
    ],
    "ConfigurationSetName": "test-configuration-set"
}
```


```bash
aws sesv2 send-bulk-email --cli-input-json file://test_bulk_email.json | jq
{
  "BulkEmailEntryResults": [
    {
      "Status": "SUCCESS",
      "MessageId": "01060197b4fc766f-6b79a5b3-8feb-4214-9a27-7cc3032e670f-000000"
    },
    {
      "Status": "SUCCESS",
      "MessageId": "01060197b4fc7671-013e46ee-843d-4e08-bbee-4983a3017875-000000"
    }
  ]
}
```

バルク送信で個別のメールが失敗した場合、個別に表示されます：
```json
{
  "BulkEmailEntryResults": [
    {
      "Status": "SUCCESS",
      "MessageId": "01060197b5143988-a27d8935-1539-4d03-b885-886b3f2903e2-000000"
    },
    {
      "Status": "MESSAGE_REJECTED",
      "Error": "Email address is not verified. The following identities failed the check in region AP-NORTHEAST-1: xxx@gmail.con"
    }
  ]
}
```

## 参考資料
- [Using templates to send personalized email with the Amazon SES API](https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-api.html#send-personalized-email-set-up-notifications) 