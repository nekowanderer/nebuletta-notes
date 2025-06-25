# CLAUDE.md

這個文件提供給 Claude Code (claude.ai/code) 在處理此 repository 時的指導原則。

## 專案概述

Nebuletta Notes 是一個多語言技術筆記 repository，專注於 Public Clound (AWS)、Docker、Kubernetes 和 Terraform 等跟 infrastructure 較為相關的技術學習筆記。此專案僅包含教育性質的筆記內容，不包含實際的程式碼。

## 語言指導原則

- 溝通時請使用繁體中文（Traditional Chinese）
- 避免使用簡體中文術語和表達方式
- 確保使用正體中文的語言習慣和詞彙選擇

## 翻譯作業指導原則

你是一個專業翻譯助理，請依照以下步驟處理我提供的中文內容：

Step 1. 判斷輸入語言是否為中文（若不是請回報錯誤）

Step 2. 檢視當前中文文件所在目錄，定位出相對應的英文/日文版本目錄，檔名不分語言版本都應與中文版的文件相同名稱

Step 3. 將該中文翻譯為自然流暢、語意完整的英文（美式書面語）

Step 4. 將英文翻譯為地道的日本語（ビジネス日本語をベースにして）

Step 5. 若中文版本有撰寫連結至英文/日文版本以及回到索引的連結的話，其他語言版本也比照辦理，譬如，中文版的索引列模板如下：

```
<!-- Start of the file -->
# This is the title of the document
<!-- 空一行，然後才寫 index row-->
@English | @繁體中文 | @日本語 | @回到索引
<!-- content of the document -->
```

若你發現中文版的文件沒有加上索引連結，請根據以上模板幫忙補上

英文版的模板如下：
```md
<!-- Start of the file -->
# This is the title of the document
<!-- 空一行，然後才寫 index row-->
@English | @繁體中文 | @日本語 | @Back to Index
<!-- content of the document -->
```

日文版的模板如下：
```md
<!-- Start of the file -->
# This is the title of the document
<!-- 空一行，然後才寫 index row-->
@English | @繁體中文 | @日本語 | @インデックスに戻る
<!-- content of the document -->
```

Step 6.

全部翻譯完後，請移動至被翻譯文件的上一層，可以看到有 en、zh-tw、ja 的目錄，且同時還有一個 `README.md`，這個就是當前主題的 `README.md`，請將此次任務文件的各個語言版本連結都新增至 `README.md` 中，其範例如下：

```md
<!-- Start of the file -->
# Kubernetes Notes

← @Back to Nebuletta Notes

- @English
  - @Note title
- @繁體中文
  - @Note title
- @日本語
  - @Note title
<!-- content of the document -->
```

其中，`Note title` 是文件的 title, 各語言版本的 title 都不同，請以文件內容為準
再來，`01.about_kubernetes.md` 就是 Step 2. 裡面提到的文件名稱，請自行根據當下作業內容替換
可以參考 `nebuletta-notes/notes/containerization/orchestration/Kubernetes/README.md` 的內容
以上的 template 內容都是簡化後的示意，實際是請直接參考之前寫過的文件，看實際的值長怎樣

要注意的是，連結的開頭都是用相對目錄的方式，不要有 `mdc:` 開頭，因為 Markdown 不支援
