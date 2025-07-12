# Docker vs. Podman アーキテクチャ比較

[English](../en/04_docker_vs_podman_architecture.md) | [繁體中文](../zh-tw/04_docker_vs_podman_architecture.md) | [日本語](../ja/04_docker_vs_podman_architecture.md) | [インデックスに戻る](../README.md)

## Docker アーキテクチャとセキュリティリスク

Docker はクライアント・サーバーアーキテクチャを採用しており、Docker Daemon がコアコンポーネントです：

- **Docker Client**：ユーザーコマンドを受信
- **Docker Daemon**：
  - **ROOT権限**で実行
  - イメージとコンテナを管理
  - システムの**単一障害点**（Single Point of Failure）
  - **セキュリティリスク**が存在（UNSAFE）

### Docker アーキテクチャ図

```
ユーザー → Docker Client → Docker Daemon (ROOT権限)
                              ↓
                         [images] [containers]
```

## Podman アーキテクチャの利点

Podman（Pod Manager）はデーモンレスアーキテクチャを採用：

- **Podman Client**：コンテナを直接管理
- **デーモン不要**：
  - バックグラウンドサービスプロセスが不要
  - 各コンテナがユーザー権限で実行
  - 単一障害点の問題なし
  - より安全な権限管理

### Podman アーキテクチャ図

```
ユーザー → Podman Client
              ↓
         [images] [containers]
```

## 主要な違いの比較

| 特徴 | Docker | Podman |
|------|--------|--------|
| アーキテクチャモード | Client-Daemon | デーモンレス |
| 権限要件 | ROOT必要 | ユーザー権限 |
| セキュリティ | 低い（特権実行） | 高い（非特権） |
| 単一障害点 | 存在（Daemon） | 存在しない |
| リソース消費 | 高い（常駐Daemon） | 低い（オンデマンド起動） |
| 基盤ランタイム | containerd | runc（よりOCI準拠ランタイムに近い） |

## セキュリティへの影響

### Docker のセキュリティ懸念
- Docker Daemon がroot権限で実行されるため、攻撃を受けるとシステム全体に影響する可能性
- すべてのコンテナが同一のDaemonを共有し、攻撃面が増加
- Daemon の障害がすべてのコンテナ実行に影響

### Podman のセキュリティ利点
- 各コンテナがユーザー権限で実行され、権限昇格リスクを軽減
- 常駐する特権プロセスなし
- コンテナ間が相対的に独立し、横方向攻撃リスクを軽減

## 実用的な考慮事項

Podman がセキュリティ面で明確な利点を持つものの、選択時には以下を考慮する必要があります：

- **互換性**：Docker エコシステムがより成熟
- **学習コスト**：Podman コマンドは Docker と高い類似性
- **エンタープライズサポート**：Docker がより包括的な商用サポート
- **ツール統合**：多くのCI/CDツールが主にDockerをサポート

## 結論

Podman のデーモンレスアーキテクチャは、セキュリティとシステム安定性において明確な利点があり、特に以下の場面に適しています：
- セキュリティ要件が高い環境
- マルチユーザーシステム
- 単一障害点を避ける必要があるシナリオ

一方、Docker は使いやすさとエコシステムの完全性において依然として優位性を保っています。