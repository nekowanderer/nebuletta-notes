# K8S 解決了哪些問題?

[English](../en/02_what_k8s_solved.md) | [繁體中文](../zh-tw/02_what_k8s_solved.md) | [日本語](../ja/02_what_k8s_solved.md) | [回到索引](../README.md)


### 1. 動態擴展（Dynamic Scaling）
K8S 透過自動擴展（Auto-scaling）功能，根據流量或運算需求自動調整應用程式的副本數量。當流量上升時，自動增加 Pod 數量；流量下降時，自動減少，確保資源利用率最佳化，同時節省成本。

### 2. 自我修復（Self Healing）
K8S 會自動監控應用程式的健康狀態。如果有 Pod 異常或失效，K8S 會自動重啟或重新建立新的 Pod 來取代異常的 Pod，確保服務持續可用，減少人為介入。

### 3. 零停機滾動更新（Zero-Downtime Rolling Update）
K8S 支援滾動更新（Rolling Update），可以逐步將舊版本的應用程式替換成新版本，過程中確保服務不中斷。這樣用戶不會感受到服務中斷，更新過程平滑且安全。

### 4. 零停機回滾（Zero-Downtime Rollback）
如果新版本部署後發現問題，K8S 可以快速回滾（Rollback）到先前穩定的版本，同樣不會造成服務中斷。這讓部署更有彈性，也降低了更新失敗的風險。
