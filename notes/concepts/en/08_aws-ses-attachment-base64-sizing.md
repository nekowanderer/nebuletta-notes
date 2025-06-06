# Practical Guide to Email Attachment Base64 Encoding and Size Calculation in AWS SES

[English](08_aws-ses-attachment-base64-sizing.md) | [繁體中文](../zh-tw/08_aws-ses-attachment-base64-sizing.md) | [日本語](../ja/08_aws-ses-attachment-base64-sizing.md) | [Back to Index](../README.md) 

## Background
During a company design review meeting discussing AWS SES settings, it was mentioned that when uploading attachments through the AWS SDK, we can skip the process of encoding attachments to base64 format as the AWS SDK handles this automatically.

While this is convenient, it's important to note that when using SES v2 API or SMTP to send emails, the maximum total size of a single message is `40MB` (this is the total size after base64 encoding).

This "maximum total size" refers to the sum of all content in the email, including:
- Email body (text, html)
- All headers
- All attachments

In other words, it's not a 40MB limit per attachment, but rather a 40MB limit for all attachments plus the email content combined. If you have multiple attachments, their total size (plus the email content) cannot exceed this limit.

However, base64 encoding is a mechanism that causes file expansion, so the actual files that can be uploaded by clients will be smaller than this limit. This note explains the reason for this expansion.

## Why does base64 encoding cause file expansion? Is the expansion ratio fixed?

Base64 is a method of encoding binary data into ASCII characters, commonly used in email attachments, JWTs, etc.

* Every 3 bytes (24 bits) are split into 4 segments of 6 bits each.
* Each 6-bit segment corresponds to 1 ASCII character (64 possible characters).
* Therefore, every 3 bytes → becomes 4 bytes, with an expansion ratio of approximately 4 ÷ 3 ≈ 1.33 times, an increase of about 33%.

The expansion ratio is almost fixed and independent of file content.

* Note that padding with `=` is used if the end is not a multiple of 3.
* Some older specifications might add line breaks every 76 characters, but modern AWS SDKs don't.

Estimation formula:

```plaintext
base64_encoded_size = ceil(original_size_in_bytes / 3) * 4
```

Or approximately:

```plaintext
base64_encoded_size ≈ original_size_in_bytes * 1.37
```

For example, if SES has a maximum email size of 40MB, you should limit the total size of original attachments in a single email to less than 30MB.

## Why does 3 bytes → 4 6-bit segments cause expansion?

Let's visualize the Base64 encoding process:

```
Original data (3 bytes = 24 bits):
+--------+--------+--------+
|  byte1 |  byte2 |  byte3 |
| 8 bits | 8 bits | 8 bits |
+--------+--------+--------+

After Base64 encoding (4 bytes = 32 bits):
+--------+--------+--------+--------+
| char1  | char2  | char3  | char4  |
| 6 bits | 6 bits | 6 bits | 6 bits |
+--------+--------+--------+--------+
```

Explanation:
1. In the original data, each byte is 8 bits and can represent any value from 0 to 255
2. During Base64 encoding, 24 bits (3 bytes) are regrouped into 4 segments of 6 bits each
3. Each 6-bit segment corresponds to a printable ASCII character
4. Although each 6-bit segment only uses 6 bits of information, it still requires a full byte (8 bits) for storage
5. Therefore, the original 24 bits (3 bytes) becomes 32 bits (4 bytes), causing approximately 33% expansion

## Why use 8 bits (1 byte) to represent a 6-bit segment?

* Each 6-bit segment corresponds to an ASCII character after encoding, and ASCII characters are at least 1 byte
* When actually stored, each byte carries 6 bits of information, with the remaining 2 bits not used for content storage but for character encoding

## Base64 = For Legal and Safe Text Transmission

* Email protocol (SMTP) is ancient and only accepts 7-bit ASCII
* If binary is transmitted directly, control characters or bytes above 127 may cause transmission errors or rejection
* Base64 converts binary → ASCII (printable, no control codes, no special symbols)
* Email/MIME protocol requires attachments to use Base64 encoding

## The 64 Characters of Base64

| Number  | Character Range | Description     |
| ------- | --------------- | --------------- |
| 0-25    | A-Z            | 26 uppercase letters |
| 26-51   | a-z            | 26 lowercase letters |
| 52-61   | 0-9            | 10 digits       |
| 62      | +              | Plus sign       |
| 63      | /              | Forward slash   |

Total of 64 characters:

```
ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/
```

Padding character: `=`

## Purpose of Padding

| Original bytes | Base64 characters | Padding |
| -------------- | ----------------- | ------- |
| 3 bytes        | 4 chars           | None    |
| 2 bytes        | 3 chars + `=`     | One     |
| 1 byte         | 2 chars + `==`    | Two     |

## Summary

Base64 sacrifices space for portability and compatibility to safely carry binary data in a text-only world.

## References
- [Service quotas in Amazon SES](https://docs.aws.amazon.com/ses/latest/dg/quotas.html)
- [Working with email attachments in SES](https://docs.aws.amazon.com/ses/latest/dg/attachments.html)
