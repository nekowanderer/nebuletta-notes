# Docker 解決了哪些問題?

[English](../en/01_what_docker_solved.md) | [繁體中文](../zh-tw/01_what_docker_solved.md) | [日本語](../ja/01_what_docker_solved.md) | [回到索引](../README.md)

```
              Why Use Docker?

   +--------+     +--------+     +---------+
   | Linux  |     |   Mac  |     | Windows |
   |(Docker)|     |(Docker)|     |(Docker) |
   +--------+     +--------+     +---------+
       |            |             |
       +------------+-------------+
                    |
              +-------------+
              |  Dockerhub  |
              +-------------+
               /           \
      +--------+           +--------+
      |  Image |           |  Image |
      +--------+           +--------+
         /                       \
+----------------+          +-----------------+
| Runtime        |          | Database        |
| Command        |          | Command         |
| Source Code    |          | Testing Data    |
+----------------+          +-----------------+

```

### 簡化部署流程

Docker 的主要功能之一是簡化應用程式的部署流程。在傳統部署方式中，我們需要：
1. 安裝特定程式語言的 Runtime 環境
2. 執行對應的啟動指令（如 Java 的 JAR 或 JavaScript 的 NPM）
3. 準備客製化的程式碼

這些步驟原本需要工程師手動整合，但 Docker 將這些元素打包成一個稱為 Image 的單一檔案，包含：
- Runtime 環境
- 啟動指令
- 程式碼

### 資料庫測試環境的快速部署

Docker 不僅適用於應用程式部署，也能用於資料庫環境的快速建立：
1. 選擇所需的資料庫（如 MySQL 或 SQL Server）
2. 準備測試資料
3. 設定啟動指令

透過 Docker Image，我們可以：
- 快速建立測試資料庫環境
- 隨時刪除重建
- 確保環境的一致性

### 跨平台部署能力

Docker 的第三個重要功能是實現跨平台部署：
1. 透過 Docker Hub 分享 Image
2. 在任何安裝了 Docker Engine 的平台上運行
3. 支援 Windows、Mac 和 Linux 等不同作業系統

### 總結

Docker 提供三大核心功能：
1. 簡化應用程式部署流程
2. 快速建立可重複使用的測試環境
3. 實現跨平台部署

這些功能讓開發和部署流程更加標準化和效率化，大幅降低了環境配置的複雜度。

