# AWS SES Quick Note

[English](en/13_aws_ses_quick_note.md) | [繁體中文](../zh-tw/13_aws_ses_quick_note.md) | [日本語](../ja/13_aws_ses_quick_note.md) | [Back to Index](../README.md)

## Creating Templates
Create a JSON format template:
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

Upload via AWS CLI:
```bash
$ aws sesv2 create-email-template --cli-input-json file://ses_template.json
```

Check if upload was successful:
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

Verify template content via command:
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

Creating duplicate templates will be blocked:
```bash
$ aws sesv2 create-email-template --cli-input-json file://ses_template.json
An error occurred (AlreadyExistsException) when calling the CreateEmailTemplate operation: Template ses_template already exists for account id 362395300803
```

Update template:
```bash
$ aws sesv2 update-email-template --cli-input-json file://ses_template.json
```

## Sending Emails to Single Destination

Create configuration set
- Navigate directly to SES -> Configuration -> Configuration sets in AWS console to create a test one, in this example we create `test-configuration-set`

Verify sender and recipient identities
- In SES -> Configuration -> Identities, create identity (domain/email address) according to your AWS SES plan and rules

Create test_email.json:

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
- TemplateName: Must match the one created in the previous step
- TemplateData: Required parameters must be included. If parameters are incorrect, emails may not be delivered, and AWS CLI won't explicitly report errors unless SNS topics are configured (which is complicated). It's usually better to validate parameters in your application first, and error handling is relatively easier to control

Send email:
```bash
aws sesv2 send-email --cli-input-json file://test_email.json | jq
{
  "MessageId": "01060197b4eb2897-6d209e61-ca60-4605-8dc7-474f3ad1eb18-000000"
}
```

## Sending Emails to Multiple Destinations

Create test_bulk_email.json for batch sending:

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

If individual emails in the bulk send fail, they will be displayed separately:
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