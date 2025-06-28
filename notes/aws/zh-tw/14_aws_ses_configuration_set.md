# 關於 AWS SES Configuration Set

[English](../en/14_aws_ses_configuration_set.md) | [繁體中文](14_aws_ses_configuration_set.md) | [日本語](../ja/14_aws_ses_configuration_set.md) | [回到索引](../README.md)

## 什麼是 Configuration Set？
Configuration Set 就是一組針對 email 發送行為的規則集合，可以把它想成「發信的設定模板」。
它讓你可以針對特定一批信件，套用指定的 tracking、log、事件通知或額外功能。

## 常見用途
- 事件追蹤（Event Tracking）
  - 你可以把open、click、bounce、complaint 等事件，自動推送到 SNS、CloudWatch Logs、Kinesis、Pinpoint
  - 例如：寄出 A 產品的行銷信用一組 config set，把所有 bounce 送到指定 SNS topic
- 分流管理
  - 不同 service、不同 type（transactional/marketing）可以指定不同 config set
  - 方便日後 log 分析、A/B 測試、異常警告
- 啟用額外功能
  - 開啟 Deliverability dashboard、IP Pool、Reputation management（適用進階用戶）
- 自動化監控與警示
  - 例如「只要某 config set 退信率超過 5%，自動發警報到 Slack」

## 實際操作方式
- 在 SES Console 建立一個 configuration set
- 可以在 config set 裡面綁定各種 destination（SNS、CloudWatch、Kinesis）
- 發信時，在 API 裡面加上這個 config set 名稱，例如：
  ```java
  SendEmailRequest.withConfigurationSetName("my-campaign-set")
  ```
- 這批信件產生的所有追蹤事件，都會根據 config set 規則處理

## 範例場景
- 有一個新活動行銷信，想追蹤所有開信點擊與退信，還想跟正常交易通知區分
- 建一個 config set promo-may2025，綁定 SNS event topic，專門推送這次活動事件
- 寄信時加上這個 config set，未來只要看 SNS/CloudWatch 就能分辨每一批信件的行為

## SES Email Sending Events
| 事件類型             | 意義                     | 重點說明                |
| ---------------- | ---------------------- | ------------------- |
| Send             | SES 收到寄信請求             | 進入發信流程，但未必成功投遞      |
| Delivery         | 信件已到對方信箱伺服器            | 最接近「送達」概念，不代表進收件匣   |
| Open             | 收件人打開信（tracking pixel） | 不是 100% 準確，有些信箱不會觸發 |
| Click            | 收件人點擊信內連結              | 衡量互動/行銷成效           |
| Bounce           | 對方退信                   | 分「硬退信/軟退信」，硬退信要移除名單 |
| Complaint        | 被標垃圾信                  | 嚴重影響 domain 信譽      |
| Reject           | SES 拒絕寄信               | 多為驗證失敗、配額用盡等        |
| RenderingFailure | 信件產生失敗                 | 變數漏、範本錯等            |
| DeliveryDelay    | 投遞延遲                   | 系統自動重試，短暫性問題        |
| Subscription     | 收件人同意接受信               | opt-in 訂閱流程，特定功能下才有 |


## 小結
- Configuration Set = 發信設定組合，可以做行為追蹤、異常監控、分流分析。
- 實務上，一個大系統往往會有多組 config set（例如：transactional、marketing、某活動專用）。
