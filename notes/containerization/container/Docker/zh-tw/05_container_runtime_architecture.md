# 容器 Runtime 架構深入解析

[English](../en/05_container_runtime_architecture.md) | [繁體中文](../zh-tw/05_container_runtime_architecture.md) | [日本語](../ja/05_container_runtime_architecture.md) | [回到索引](../README.md)

## 容器 runtime 層次架構

容器 runtime 採用分層架構，各層職責明確分離，形成一個完整的容器生態系統。

### 架構層次圖

```
使用者層面
├── Docker CLI / Podman CLI / Kubernetes
│
高層級 runtime 層
├── Docker Engine
├── containerd 
├── CRI-O
│
低層級 runtime 層
├── runc (OCI 參考實現)
├── crun (C 語言實現，更快速)
│
作業系統核心
└── Linux Kernel (namespaces, cgroups, mounts, seccomp)
```

## 各組件詳細說明

### 低層級 Runtime（Low-Level Runtime）

#### runc
- **定義**：OCI (Open Container Initiative)  runtime 規範的參考實現
- **功能**：
  - 輕量級 CLI 工具
  - 負責最終建立和啟動容器程序
  - 直接與 Linux 核心互動（namespaces、cgroups 等）
- **重要性**：所有容器平台（Docker、Podman、Kubernetes）最終都依賴 runc

#### crun（runc 的替代方案）
- **特色**：用 C 語言撰寫的高效能實作
- **優勢**：
  - 體積比 runc 小 50 倍
  - 執行速度比 runc 快 2 倍
  - 記憶體佔用更低
- **推薦**：由於性能優勢，被推薦作為容器 runtime 

### 高層級 runtime（High-Level Runtime）

#### containerd
- **位置**：位於 runc 之上的管理層
- **職責**：
  - 映像管理（拉取、儲存、分發）
  - 容器生命週期管理
  - 網路配置
  - 儲存管理
- **機制**：通過 containerd-shim 進程委託給 runc 執行
- **標準**：完全支援 OCI 規範

#### Docker Engine
- **定位**：建構在 containerd 之上的完整解決方案
- **提供**：
  - 完整的開發者體驗
  - 豐富的工具鏈
  - API 和 CLI 介面
  - Docker Compose 等高級功能

#### CRI-O
- **用途**：專為 Kubernetes 設計的 runtime 
- **特點**：
  - 實現 Container Runtime Interface (CRI)
  - 輕量級，專注於 Kubernetes 整合
  - 委託執行給 runc 或其他 OCI  runtime 

## Docker vs. Podman  runtime 架構比較

### Docker 的 runtime 流程

```
Docker CLI → Docker Daemon → containerd → containerd-shim → runc → 容器程序
```

**特點**：
- 使用 containerd 作為高層級 runtime 
- 需要 Docker Daemon 持續運行
- 集中式管理，所有容器共享同一個 daemon
- 多層架構提供豐富功能

### Podman 的 runtime 流程

```
Podman CLI → runc/crun → 容器程序
```

**特點**：
- 直接使用 runc/crun，跳過高層級 runtime 
- 無需 daemon，每個容器獨立運行
- 更接近 OCI 規範的原生實現
- 簡化架構，減少中間層

## OCI 合規性深入分析

### 什麼是 OCI（Open Container Initiative）

OCI 是容器技術的開放標準，定義了：
- **Runtime 規範**：如何運行容器
- **映像規範**：容器映像格式
- **分發規範**：映像分發標準

### 為什麼 Podman 更接近 OCI-compliant？

#### 1. 直接 OCI 整合
- Podman 直接使用 runc/crun，減少中間層
- 更純粹的 OCI  runtime 實現
- 避免額外的抽象層

#### 2. 無 Daemon 架構
- 符合 OCI 規範的簡潔設計理念
- 每個容器作為獨立程序運行
- 降低系統複雜度

#### 3. 標準化程度
- 更嚴格遵循 OCI 容器映像格式
- 與其他 OCI 相容工具互操作性更好
- 更好的跨平台相容性

## Kubernetes 整合架構

### Container Runtime Interface (CRI)

```
Kubernetes → CRI → High-Level Runtime → Low-Level Runtime
```

**支援的 runtime**：
- **containerd**：通過 CRI 插件
- **CRI-O**：原生 CRI 實現
- **Docker**：通過 dockershim（已被移除）

### kubelet 與 runtime 互動

1. **容器創建請求**：kubelet 向 CRI 發送請求
2. **映像管理**：高層級 runtime 處理映像拉取
3. **容器啟動**：委託給低層級 runtime 執行
4. **狀態監控**：返回容器狀態給 kubelet

## 性能與安全性比較

### 性能表現

| 特性 | Docker | Podman | 原因 |
|------|--------|--------|------|
| 啟動速度 | 較慢 | 較快 | 無 daemon 開銷 |
| 記憶體使用 | 較高 | 較低 | 無常駐程序 |
| 資源消耗 | 高 | 低 | 簡化架構 |

### 安全性考量

#### Docker 安全特點
- 需要特權 daemon，增加攻擊面
- 集中式管理，單點故障風險
- 豐富的安全工具和插件

#### Podman 安全優勢
- rootless 執行，降低權限風險
- 分散式架構，減少攻擊面
- 無常駐特權程序

## 選擇建議

### 選擇 Docker 的情況
- 需要完整的開發者生態系統
- 使用 Docker Compose 等高級工具
- 團隊已熟悉 Docker 工作流程
- 需要商業支援

### 選擇 Podman 的情況
- 安全要求較高的環境
- 希望更接近 OCI 標準
- 需要 rootless 容器執行
- 系統資源有限的環境

## 未來發展趨勢

### 標準化趨勢
- OCI 規範持續演進
-  runtime 之間互操作性提升
- 更多 OCI 相容工具出現

### 技術發展
- **crun** 等高效能 runtime 普及
- **Rootless** 容器成為主流
- **微虛擬機** runtime （如 Kata Containers）整合

## 總結

容器 runtime 架構展現了現代容器技術的分層設計理念。理解這些架構差異有助於：

- 選擇適合的容器平台
- 優化容器部署策略
- 提升系統安全性和性能
- 更好地進行故障排除和調優

無論選擇 Docker 還是 Podman，了解底層 runtime 機制都是容器技術掌握的關鍵。
