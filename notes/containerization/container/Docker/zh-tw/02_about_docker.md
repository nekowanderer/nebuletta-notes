# 關於 Docker

[English](../en/02_about_docker.md) | [繁體中文](../zh-tw/02_about_docker.md) | [日本語](../ja/02_about_docker.md) | [回到索引](../README.md)

### Docker 基本架構
Docker 的基本架構包含以下幾個重要元件：
1. Docker Engine
   - 提供標準化的容器環境
   - 負責管理系統資源分配（CPU、記憶體等）
   - 作為容器運行的基礎平台

2. Docker Container
   - 虛擬化的運行空間
   - 透過 Docker Engine 獲取系統資源
   - 可以同時運行多個 Container

3. Docker Image
   - 包含應用程式所需的環境與檔案
   - 可基於現有 Image（如 CentOS）進行客製化
   - 是容器化部署的最小單位

### 跨平台部署機制
Docker 實現跨平台部署的關鍵在於：
1. 標準化環境
   - 在任何作業系統（Linux、Windows、MacOS）上安裝 Docker Engine
   - 所有環境都遵循相同的容器化標準

2. Docker Hub
   - 雲端空間用於分享 Docker Image
   - 開發者可以上傳自製的 Image
   - 其他使用者可以下載並使用這些 Image
   - 實現全球開發者之間的資源共享

### 實務操作架構
在實際操作中，Docker 的工作流程如下：

1. 開發階段
   - 撰寫 Dockerfile 定義 Image 規格
   - 使用 `docker build` 指令建立 Image
   - 使用 `docker images` 查看現有 Image

2. 運行階段
   - 使用 `docker run` 啟動 Container
   - 使用 `docker container ls` 監控運行中的 Container
   - 透過 `-p` 參數設定網路連接埠
   - 使用 `docker network ls` 查看網路設定

3. 資料持久化
   - 使用 Docker Volume 進行資料儲存
   - 資料存在於 Container 外部
   - 使用 `-v` 參數掛載 Volume
   - 使用 `docker volume ls` 查看現有 Volume

### 圖解
1. Container 預設是隔離的，需要特別設定才能對外開放
2. Volume 的設計確保資料不會因 Container 刪除而遺失
3. Docker Engine 是整個架構的核心，負責協調所有元件運作

```
+----------------------+
|        Host          |
+----------------------+
|     Docker Engine    |
+----------------------+
|      Network         | <--- docker run -p / docker network ls
+----------------------+
|      Container       | <--- docker run / docker container ls
|   +--------------+   |
|   |   Volume     |   | <--- docker run -v / docker volume ls
|   +--------------+   |
+----------------------+
|      Image           | <--- docker build / docker images
|   +------------+     |
|   | Dockerfile |     |
|   +------------+     |
+----------------------+
```

#### 說明
- 這張圖是從 Docker image 的角度往上一層一層看
- 每一層代表 Docker 架構中的一個元件，從底層的 Image（內含 Dockerfile）到最上層的 Host。
- Volume 是 Container 內部的一個資料儲存區塊，與外部資料持久化有關。
- Docker Engine 是 Network 與 Host 之間的核心層，負責協調所有元件運作。
- 右側標註了常用指令，對應每一層的操作與查詢。

根據圖表可知，Docker 的每個元件都可以透過對應的指令進行管理與查詢，這有助於理解 Docker 的分層設計與實務操作流程。
