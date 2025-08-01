# About AWS SES Configuration Set

[English](14_aws_ses_configuration_set.md) | [繁體中文](../zh-tw/14_aws_ses_configuration_set.md) | [日本語](../ja/14_aws_ses_configuration_set.md) | [Back to Index](../README.md)

## What is a Configuration Set?
A Configuration Set is a collection of rules for email sending behavior, which can be thought of as an "email sending template." It allows you to apply specific tracking, logging, event notifications, or additional features to a particular batch of emails.

## Common Use Cases
- Event Tracking
  - You can automatically push events such as opens, clicks, bounces, complaints, etc., to SNS, CloudWatch Logs, Kinesis, or Pinpoint
  - For example: Use one config set for marketing emails for Product A, sending all bounces to a specified SNS topic
- Traffic Management
  - Different services and different types (transactional/marketing) can be assigned different config sets
  - Facilitates future log analysis, A/B testing, and anomaly alerts
- Enable Additional Features
  - Enable Deliverability dashboard, IP Pool, Reputation management (for advanced users)
- Automated Monitoring and Alerts
  - For example: "Automatically send alerts to Slack when bounce rate exceeds 5% for a specific config set"

## Practical Implementation
- Create a configuration set in the SES Console
- You can bind various destinations (SNS, CloudWatch, Kinesis) within the config set
- When sending emails, add the config set name to the API, for example:
  ```java
  SendEmailRequest.withConfigurationSetName("my-campaign-set")
  ```
- All tracking events generated by this batch of emails will be processed according to the config set rules

## Example Scenario
- For a new marketing campaign, you want to track all opens, clicks, and bounces, and distinguish them from regular transaction notifications
- Create a config set called "promo-may2025" and bind it to an SNS event topic specifically for pushing campaign events
- Add this config set when sending emails, so in the future you can distinguish the behavior of each batch of emails by looking at SNS/CloudWatch

## SES Email Sending Events
| Event Type        | Meaning                    | Key Points                    |
| ----------------- | -------------------------- | ---------------------------- |
| Send              | SES receives send request   | Enters sending process, but may not be successfully delivered |
| Delivery          | Email reaches recipient's mail server | Closest to "delivered" concept, doesn't mean it reached inbox |
| Open              | Recipient opens email (tracking pixel) | Not 100% accurate, some mailboxes don't trigger |
| Click             | Recipient clicks link in email | Measures engagement/marketing effectiveness |
| Bounce            | Recipient bounces email     | Divided into "hard bounce/soft bounce," hard bounces should be removed from list |
| Complaint         | Marked as spam              | Seriously affects domain reputation |
| Reject            | SES rejects sending email   | Usually due to verification failure, quota exhaustion, etc. |
| RenderingFailure  | Email generation failure    | Missing variables, template errors, etc. |
| DeliveryDelay     | Delivery delay              | System automatically retries, temporary issues |
| Subscription      | Recipient agrees to receive emails | opt-in subscription process, only available under specific features |

## Summary
- Configuration Set = Email sending configuration combination that enables behavior tracking, anomaly monitoring, and traffic analysis.
- In practice, a large system often has multiple config sets (e.g., transactional, marketing, campaign-specific). 