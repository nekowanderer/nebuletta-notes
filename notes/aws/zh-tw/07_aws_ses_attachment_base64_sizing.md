# 從 AWS SES 實務談 Email 附件 Base64 編碼與容量計算

[English](../en/07_aws_ses_attachment_base64_sizing.md) | [繁體中文](./07_aws_ses_attachment_base64_sizing.md) | [日本語](../ja/07_aws_ses_attachment_base64_sizing.md) | [回到索引](../README.md)

## 背景
某天在公司的 design review meeting 裡，提到一些跟 AWS SES 有關的設定，其中，在上傳附加檔案的時候，如果是透過 AWS SDK 來處理的話，我們可以省去將附加檔案編碼為 base64-encoding 格式的過程，因為 AWS SDK 會自動處理這段。

這聽起來很方便，但是要注意使用 SES v2 API 或 SMTP 發信時，單一封信（message）最大總大小為 `40MB`（這是 base64-encoding 後的總大小）。

這個「最大總大小」是指整封信的所有內容加總，包含：
- 郵件主體（text、html）
- 所有 header
- 所有附件（attachments）

也就是說，不是單一個附件最大 40MB，而是所有附件加上信件內容總和最大 40MB。如果你有多個附件，全部加起來（再加上信件本身內容）不能超過這個限制。

然而，base64-encoding 是會讓檔案膨脹的一種機制，所以實際上客戶端可以上傳的檔案會比實際的上限要來得更小，這篇筆記會解釋膨脹的原因。

## 為什麼 base64 encoding 之後檔案會變大？膨脹比例固定嗎？

Base64 是一種將二進位資料轉為 ASCII 字元的編碼方式，常見於 email 附件、JWT 等。

* 每 3 個 byte（24 bits）會被拆成 4 個 6-bit 的區段。
* 每個 6-bit 區段對應 1 個 ASCII 字元（共 64 種字元）。
* 因此，原本每 3 byte → 變成 4 byte，膨脹比例約為 4 ÷ 3 ≈ 1.33 倍，增加約 33%。

膨脹比例幾乎固定，和檔案內容無關。

* 須注意尾端若不是 3 的倍數會用 `=` 做 padding。
* 某些舊規範可能會每 76 字元換行，但現代 AWS SDK 不會。

估算公式：

```plaintext
base64_encoded_size = ceil(original_size_in_bytes / 3) * 4
```

或近似：

```plaintext
base64_encoded_size ≈ original_size_in_bytes * 1.37
```

例如 SES 若最大 email size 為 40MB，應限制單封 email 的原始附件總大小小於 30MB。


## 3 個 bytes → 4 個 6 bits，為什麼會膨脹？

讓我們用視覺化的方式來理解 Base64 編碼的膨脹過程：

```
原始資料（3 bytes = 24 bits）：
+--------+--------+--------+
|  byte1 |  byte2 |  byte3 |
| 8 bits | 8 bits | 8 bits |
+--------+--------+--------+

Base64 編碼後（4 bytes = 32 bits）：
+--------+--------+--------+--------+
| char1  | char2  | char3  | char4  |
| 6 bits | 6 bits | 6 bits | 6 bits |
+--------+--------+--------+--------+
```

說明：
1. 原始資料中，每個 byte 是 8 bits，可以表示 0~255 的任意值
2. Base64 編碼時，將 24 bits（3 bytes）重新分組為 4 個 6-bit 區段
3. 每個 6-bit 區段對應一個 ASCII 可列印字元
4. 雖然每個 6-bit 區段只使用 6 bits 的資訊，但在儲存時仍然需要一個完整的 byte（8 bits）
5. 因此，原本的 24 bits（3 bytes）變成了 32 bits（4 bytes），造成約 33% 的膨脹


## 6 bit 區段為何要用 8 bit（1 byte）來表現？

* 每個 6bit 編碼後對應一個 ASCII 字元，ASCII 字元最小也是 1 byte。
* 實際存放時，仍然是用一個 byte 承載 6bit 資訊，其餘 2bit 沒被用來儲存內容，而是當作字元編碼。


## Base64 = 為了合法、安全的文字傳輸

* Email 協定（SMTP）設計古老，只接受 7-bit ASCII
* 若直接傳 binary，會遇到控制字元或超過 127 的 byte，容易在傳送過程出錯、被拒收
* Base64 將 binary → ASCII（可列印、無控制碼、無特殊符號）
* Email/MIME 協定規定附件必須用 Base64 encoding


## Base64 的 64 個字元

| 編號    | 字元區間 | 說明        |
| ------ | ---- | --------- |
| 0\~25  | A\~Z | 英文大寫 26 個 |
| 26\~51 | a\~z | 英文小寫 26 個 |
| 52\~61 | 0\~9 | 數字 10 個   |
| 62     | +    | 加號        |
| 63     | /    | 斜線        |

總共 64 個字元：

```
ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/
```

Padding 字元：`=`


## Padding 的用途

| 原始 byte 數 | Base64 字元數   | Padding |
| ----------- | -------------- | ------- |
| 3 byte      | 4 chars        | 無       |
| 2 byte      | 3 chars + `=`  | 有一個     |
| 1 byte      | 2 chars + `==` | 有兩個     |


## 小結語

Base64 是為了在純文字世界安全攜帶 binary 資料，犧牲空間換取可攜性與兼容性。


## 參考文獻
- [Service quotas in Amazon SES](https://docs.aws.amazon.com/ses/latest/dg/quotas.html)
- [Working with email attachments in SES](https://docs.aws.amazon.com/ses/latest/dg/attachments.html)
