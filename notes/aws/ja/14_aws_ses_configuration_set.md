# AWS SES Configuration Set について

[English](../en/14_aws_ses_configuration_set.md) | [繁體中文](../zh-tw/14_aws_ses_configuration_set.md) | [日本語](14_aws_ses_configuration_set.md) | [インデックスに戻る](../README.md)

## Configuration Set とは？
Configuration Set は、メール送信行動に関するルールの集合体で、「メール送信の設定テンプレート」と考えることができます。特定のメールバッチに対して、指定されたトラッキング、ログ、イベント通知、または追加機能を適用することができます。

## 一般的な用途
- イベント追跡（Event Tracking）
  - open、click、bounce、complaint などのイベントを SNS、CloudWatch Logs、Kinesis、Pinpoint に自動的にプッシュできます
  - 例：A 製品のマーケティングメールに一つの config set を使用し、すべての bounce を指定された SNS topic に送信
- トラフィック管理
  - 異なるサービス、異なるタイプ（transactional/marketing）に異なる config set を割り当てることができます
  - 将来のログ分析、A/B テスト、異常警告に便利
- 追加機能の有効化
  - Deliverability dashboard、IP Pool、Reputation management を有効化（上級ユーザー向け）
- 自動化された監視とアラート
  - 例：「特定の config set の bounce 率が 5% を超えた場合、Slack に自動的にアラートを送信」

## 実際の実装方法
- SES Console で configuration set を作成
- config set 内で様々な destination（SNS、CloudWatch、Kinesis）をバインドできます
- メール送信時に、API にこの config set 名を追加します。例：
  ```java
  SendEmailRequest.withConfigurationSetName("my-campaign-set")
  ```
- このメールバッチで生成されるすべての追跡イベントは、config set ルールに従って処理されます

## 例のシナリオ
- 新しいキャンペーンのマーケティングメールで、すべての開封、クリック、bounce を追跡し、通常のトランザクション通知と区別したい
- promo-may2025 という config set を作成し、このキャンペーンイベント専用の SNS event topic にバインド
- メール送信時にこの config set を追加し、将来は SNS/CloudWatch を見るだけで各メールバッチの行動を区別できます

## SES Email Sending Events
| イベントタイプ        | 意味                        | 重要なポイント                    |
| ------------------- | --------------------------- | ------------------------------ |
| Send                | SES が送信リクエストを受信      | 送信プロセスに入るが、必ずしも成功して配信されるとは限らない |
| Delivery            | メールが受信者のメールサーバーに到達 | 「配信」概念に最も近いが、受信トレイに到達したことを意味しない |
| Open                | 受信者がメールを開封（tracking pixel） | 100% 正確ではなく、一部のメールボックスはトリガーしない |
| Click               | 受信者がメール内のリンクをクリック | エンゲージメント/マーケティング効果を測定 |
| Bounce              | 受信者がメールを bounce       | 「ハード bounce/ソフト bounce」に分かれ、ハード bounce はリストから削除すべき |
| Complaint           | スパムとしてマークされる        | ドメインの評判に深刻な影響を与える |
| Reject              | SES がメール送信を拒否        | 主に認証失敗、クォータ枯渇などが原因 |
| RenderingFailure    | メール生成失敗               | 変数不足、テンプレートエラーなど |
| DeliveryDelay       | 配信遅延                     | システムが自動的にリトライ、一時的な問題 |
| Subscription        | 受信者がメール受信に同意        | opt-in 購読プロセス、特定機能でのみ利用可能 |

## まとめ
- Configuration Set = 行動追跡、異常監視、トラフィック分析を可能にするメール送信設定の組み合わせ
- 実際には、大規模システムには複数の config set があることが多い（例：transactional、marketing、特定キャンペーン専用） 