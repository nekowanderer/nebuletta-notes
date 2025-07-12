# Docker vs. Podman 架構比較

[English](../en/04_docker_vs_podman_architecture.md) | [繁體中文](../zh-tw/04_docker_vs_podman_architecture.md) | [日本語](../ja/04_docker_vs_podman_architecture.md) | [回到索引](../README.md)

## Docker 架構與安全風險

Docker 採用 Client-Server 架構，其中 Docker Daemon 是核心組件：

- **Docker Client**：接收使用者指令
- **Docker Daemon**：
  - 以 **ROOT 權限**運行
  - 管理 images 和 containers
  - 是系統的**單點故障**（Single Point of Failure）
  - 存在**安全風險**（UNSAFE）

### Docker 架構圖示

```
使用者 → Docker Client → Docker Daemon (ROOT權限)
                            ↓
                       [images] [containers]
```

## Podman 架構優勢

Podman（Pod Manager）採用 daemonless 架構：

- **Podman Client**：直接管理容器
- **無需 Daemon**：
  - 不需要背景服務程序
  - 每個容器以使用者權限運行
  - 沒有單點故障問題
  - 更安全的權限管理

### Podman 架構圖示

```
使用者 → Podman Client
            ↓
       [images] [containers]
```

## 關鍵差異比較

| 特性 | Docker | Podman |
|------|--------|--------|
| 架構模式 | Client-Daemon | Daemonless |
| 權限需求 | 需要 ROOT | 使用者權限 |
| 安全性 | 較低（特權運行） | 較高（無特權） |
| 單點故障 | 存在（Daemon） | 不存在 |
| 資源消耗 | 較高（常駐 Daemon） | 較低（按需啟動） |
| 底層運作環境 | containerd | runc (更接近 OCI-compliant runtime) |

## 安全性影響

### Docker 的安全疑慮
- Docker Daemon 以 root 權限運行，若被攻擊可能影響整個系統
- 所有容器共享同一個 Daemon，增加攻擊面
- Daemon 故障會影響所有容器運行

### Podman 的安全優勢
- 每個容器以使用者權限運行，降低權限提升風險
- 沒有常駐的特權程序
- 容器之間相對獨立，減少橫向攻擊風險

## 實務考量

雖然 Podman 在安全性上有明顯優勢，但選擇時仍需考慮：

- **相容性**：Docker 生態系統更成熟
- **學習成本**：Podman 指令與 Docker 高度相似
- **企業支援**：Docker 有更完整的商業支援
- **工具整合**：許多 CI/CD 工具主要支援 Docker

## 結論

Podman 的 daemonless 架構在安全性和系統穩定性方面具有明顯優勢，特別適合：
- 安全要求較高的環境
- 多使用者系統
- 需要避免單點故障的場景

而 Docker 仍在易用性和生態系統完整性方面保持領先地位。