# CLAUDE.md

這個文件提供給 Claude Code (claude.ai/code) 在處理此 repository 時的指導原則。

## 專案概述

Nebuletta Notes 是一個多語言技術筆記 repository，專注於 Public Clound (AWS)、Docker、Kubernetes 和 Terraform 等跟 infrastructure 較為相關的技術學習筆記。此專案僅包含教育性質的筆記內容，不包含實際的程式碼。

## 翻譯作業指導原則

### 翻譯流程
當你收到翻譯請求時，請依照以下步驟處理：

1. **判斷輸入語言**：確認輸入內容是否為中文（繁體中文）
2. **定位對應目錄**：找到相對應的英文 (`en`) 和日文 (`ja`) 版本目錄
3. **執行翻譯**：
   - 中文 → 英文（美式書面語）
   - 英文 → 日文（ビジネス日本語をベースにして）
4. **更新索引連結**：確保所有語言版本都有正確的導航連結
5. **更新 README.md**：在主題目錄的 README.md 中新增文件連結

### 文件結構模式
```
topic/
├── README.md (語言索引)
├── en/ (英文內容)
├── zh-tw/ (繁體中文內容)
├── ja/ (日文內容)
└── images/ (共用圖片)
```

### 索引連結模板

**繁體中文版：**
```markdown
[English](../en/filename.md) | [繁體中文](../zh-tw/filename.md) | [日本語](../ja/filename.md) | [回到索引](../README.md)
```

**英文版：**
```markdown
[English](../en/filename.md) | [繁體中文](../zh-tw/filename.md) | [日本語](../ja/filename.md) | [Back to Index](../README.md)
```

**日文版：**
```markdown
[English](../en/filename.md) | [繁體中文](../zh-tw/filename.md) | [日本語](../ja/filename.md) | [インデックスに戻る](../README.md)
```

### README.md 更新模板
```markdown
- [English](en/)
  - [Serial number. Note title](en/filename.md)
- [繁體中文](zh-tw/)
  - [Serial number. Note title](zh-tw/filename.md)
- [日本語](ja/)
  - [Serial number. Note title](ja/filename.md)
```

範例
```
- [English](en/)
  - [01. About Kubernetes](en/01_about_kubernetes.md)
  - [02. What Problems Does K8S Solve?](en/02_what_k8s_solved.md)
  - [03. Overview of Kubernetes Architecture](en/03_k8s_architecture_overview.md)
```

## 重要提醒

- 所有連結都使用相對路徑，不要使用 `mdc:` 開頭
- 翻譯時保持技術術語的一致性
- 確保各語言版本的標題準確反映內容
- 維持文件結構的一致性

## 語言指導原則

- 溝通時請使用繁體中文（Traditional Chinese）
- 避免使用簡體中文術語和表達方式
- 確保使用正體中文的語言習慣和詞彙選擇
