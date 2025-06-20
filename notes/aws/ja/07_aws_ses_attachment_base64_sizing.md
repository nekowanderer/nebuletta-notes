# AWS SES 実務から見るメール添付ファイルのBase64エンコーディングと容量計算

[English](../en/07_aws_ses_attachment_base64_sizing.md) | [繁體中文](../zh-tw/07_aws_ses_attachment_base64_sizing.md) | [日本語](07_aws_ses_attachment_base64_sizing.md) | [インデックスに戻る](../README.md) 

## 背景
会社のデザインレビュー会議でAWS SESの設定について議論した際、AWS SDKを使用して添付ファイルをアップロードする場合、Base64エンコーディングのプロセスを省略できることが話題になりました。これはAWS SDKが自動的に処理してくれるためです。

これは便利ですが、SES v2 APIまたはSMTPを使用してメールを送信する場合、1通のメッセージの最大合計サイズは`40MB`（Base64エンコーディング後の合計サイズ）であることに注意が必要です。

この「最大合計サイズ」とは、メールのすべてのコンテンツの合計を指します：
- メール本文（テキスト、HTML）
- すべてのヘッダー
- すべての添付ファイル

つまり、添付ファイル1つあたり40MBという制限ではなく、すべての添付ファイルとメールコンテンツを合わせた合計が40MBという制限です。複数の添付ファイルがある場合、それらの合計サイズ（メールコンテンツを含む）はこの制限を超えることはできません。

ただし、Base64エンコーディングはファイルを拡張するメカニズムであるため、クライアントが実際にアップロードできるファイルはこの制限よりも小さくなります。このメモでは、その拡張の理由について説明します。

## Base64エンコーディングでファイルが拡張される理由は？拡張率は固定ですか？

Base64はバイナリデータをASCII文字に変換する方法で、メール添付ファイルやJWTなどで一般的に使用されています。

* 3バイト（24ビット）ごとに4つの6ビットセグメントに分割されます。
* 各6ビットセグメントは1つのASCII文字（64種類の文字）に対応します。
* したがって、3バイト→4バイトとなり、拡張率は約4 ÷ 3 ≈ 1.33倍、約33%の増加となります。

拡張率はほぼ固定で、ファイルの内容には依存しません。

* 末尾が3の倍数でない場合は`=`でパディングされます。
* 古い仕様では76文字ごとに改行を入れる場合がありますが、最新のAWS SDKでは行われません。

推定式：

```plaintext
base64_encoded_size = ceil(original_size_in_bytes / 3) * 4
```

または概算：

```plaintext
base64_encoded_size ≈ original_size_in_bytes * 1.37
```

例えば、SESの最大メールサイズが40MBの場合、1通のメールの元の添付ファイルの合計サイズは30MB未満に制限する必要があります。

## 3バイト→4つの6ビットセグメントでなぜ拡張されるのか？

Base64エンコーディングのプロセスを視覚化してみましょう：

```
元のデータ（3バイト = 24ビット）：
+--------+--------+--------+
|  byte1 |  byte2 |  byte3 |
| 8ビット | 8ビット | 8ビット |
+--------+--------+--------+

Base64エンコーディング後（4バイト = 32ビット）：
+--------+--------+--------+--------+
| char1  | char2  | char3  | char4  |
| 6ビット | 6ビット | 6ビット | 6ビット |
+--------+--------+--------+--------+
```

説明：
1. 元のデータでは、各バイトは8ビットで、0から255までの任意の値を表現できます
2. Base64エンコーディングでは、24ビット（3バイト）を4つの6ビットセグメントに再グループ化します
3. 各6ビットセグメントは印刷可能なASCII文字に対応します
4. 各6ビットセグメントは6ビットの情報しか使用しませんが、保存には完全なバイト（8ビット）が必要です
5. したがって、元の24ビット（3バイト）が32ビット（4バイト）になり、約33%の拡張が発生します

## なぜ6ビットセグメントを8ビット（1バイト）で表現するのか？

* 各6ビットセグメントはエンコーディング後にASCII文字に対応し、ASCII文字は少なくとも1バイト必要です
* 実際の保存時、各バイトは6ビットの情報を運び、残りの2ビットはコンテンツの保存には使用されず、文字エンコーディングに使用されます

## Base64 = 合法かつ安全なテキスト伝送のため

* メールプロトコル（SMTP）は古く、7ビットASCIIのみを受け付けます
* バイナリを直接送信すると、制御文字や127を超えるバイトが伝送エラーや拒否の原因となる可能性があります
* Base64はバイナリ→ASCII（印刷可能、制御コードなし、特殊記号なし）に変換します
* メール/MIMEプロトコルでは添付ファイルにBase64エンコーディングが必要です

## Base64の64文字

| 番号    | 文字範囲 | 説明        |
| ------- | -------- | ----------- |
| 0-25    | A-Z      | 大文字26文字 |
| 26-51   | a-z      | 小文字26文字 |
| 52-61   | 0-9      | 数字10文字   |
| 62      | +        | プラス記号   |
| 63      | /        | スラッシュ   |

合計64文字：

```
ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/
```

パディング文字：`=`

## パディングの目的

| 元のバイト数 | Base64文字数 | パディング |
| ----------- | ------------ | -------- |
| 3バイト      | 4文字        | なし     |
| 2バイト      | 3文字 + `=`  | 1つ     |
| 1バイト      | 2文字 + `==` | 2つ     |

## まとめ

Base64は、テキストのみの世界でバイナリデータを安全に運ぶために、スペースを犠牲にして移植性と互換性を確保します。

## 参考文献
- [Service quotas in Amazon SES](https://docs.aws.amazon.com/ses/latest/dg/quotas.html)
- [Working with email attachments in SES](https://docs.aws.amazon.com/ses/latest/dg/attachments.html)
