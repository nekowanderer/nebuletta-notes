# AWS SES 筆記

[English](../en/13_aws_ses_quick_note.md) | [繁體中文](zh-tw/13_aws_ses_quick_note.md) | [日本語](../ja/13_aws_ses_quick_note.md) | [回到索引](../README.md)

## 建立 Template
建立一個 JSON 格式的 template:
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

透過 AWS CLI 上傳：
```bash
$ aws sesv2 create-email-template --cli-input-json file://ses_template.json
```

檢查是否上傳成功：
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

透過指令確認 template 內容：
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

重複建立 template 會被擋掉：
```bash
$ aws sesv2 create-email-template --cli-input-json file://ses_template.json
An error occurred (AlreadyExistsException) when calling the CreateEmailTemplate operation: Template ses_template already exists for account id 362395300803
```

更新 template：
```bash
$ aws sesv2 update-email-template --cli-input-json file://ses_template.json
```

## 寄送郵件至單一 destination

建立 configuration set
- 直接在 AWS console 裡面導航至 SES -> Configuration -> Configuration sets 建立一個測試用的，此處範例建立的是 `test-configuration-set`

確認寄件人與收件人的 identity
- 在 SES -> Configuration -> Identities，根據自己的 AWS SES 方案還有規則建立 identity（domain / email address）

建立 test_email.json：

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
- TemplateName: 要跟前一步驟建立的一樣
- TemplateData: 該帶的參數要帶到，參數帶錯的話會不會送達，而且 AWS CLI 不會顯式地報錯，除非去 SNS 設定 topic（很麻煩），通常在自己的 application 裡面先對參數進行比對比較好，且錯誤處理流程相對好控制

寄送郵件：
```bash
aws sesv2 send-email --cli-input-json file://test_email.json | jq
{
  "MessageId": "01060197b4eb2897-6d209e61-ca60-4605-8dc7-474f3ad1eb18-000000"
}
```

## 寄送郵件至多重 destination

建立批次寄送的 test_bulk_email.json：

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

如果 bulk 中有各別 email 出錯的話，也會獨立顯示出來：
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

## Reference
- [Using templates to send personalized email with the Amazon SES API](https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-api.html#send-personalized-email-set-up-notifications)
