# Kubernetes HPA（Horizontal Pod Autoscalin）簡介

[English](../en/32_k8s_hpa_intro.md) | [繁體中文](../zh-tw/32_k8s_hpa_intro.md) | [日本語](../ja/32_k8s_hpa_intro.md) | [回到索引](../README.md)

Kubernetes HPA 會根據資源使用情況（如 CPU 使用率）自動調整 Pod 數量，確保應用程式能彈性擴展或縮減。其運作流程如下：

- HPA 會持續監控 Deployment 內所有 Pod 的 CPU 使用率。
- 這些資源使用數據是由 Metric Server 負責收集。Metric Server 會定期從每個 Pod 收集 CPU（或記憶體等）使用資訊，並將這些數據提供給 HPA。
- HPA 取得這些即時的監控數據後，會根據預設的門檻值（如目標 CPU 使用率）來判斷是否需要調整 ReplicaSet 中 Pod 的數量（例如從 3 個 Pod 增加或減少）。
- 當負載上升時，HPA 會自動擴增 Pod 數量；當負載下降時，則會自動縮減，提升資源利用率與服務穩定性。

## 重要元件說明
- **Cluster**：Kubernetes 叢集環境。
- **Deployment**：管理應用程式的部署與升級。
- **ReplicaSet**：確保指定數量的 Pod 持續運作。
- **Pod**：應用程式運行的最小單位，內含一個或多個 Container。
- **Metric Server**：負責收集並提供叢集中各 Pod 的即時資源使用數據，供 HPA 參考。
- **HPA**：根據 Metric Server 提供的數據，自動調整 Pod 數量。

## Metric Server 與 Prometheus 的比較

- **Metric Server**：Kubernetes 內建的輕量級資源監控元件，主要提供 HPA 及其他內部元件即時的 CPU、記憶體等基本指標。安裝簡單，適合只需要基本自動擴縮功能的場景。
- **Prometheus**：功能更強大的監控系統，不僅能收集更細緻、更多元的指標（如自訂應用程式指標），還能長期儲存數據、支援複雜查詢與告警。透過安裝如 kube-prometheus-stack 或自訂 Adapter，可以讓 HPA 直接從 Prometheus 取得指標，進行更進階的自動擴縮。

**總結**：
如果只需要基本的 HPA 功能，使用 Metric Server 即可；若有更進階的監控與自動擴縮需求，則可考慮用 Prometheus 取代 Metric Server，並搭配自訂指標進行彈性調整。

## Metrics Server 的 Pull 與 Push 模型

Kubernetes 內建的 Metric Server 以及 Prometheus 這類主流監控系統，主要採用「拉取（Pull）」模型。也就是說，Prometheus 會主動定期向各個 Pod 或服務的 `/metrics` HTTP endpoint 發出請求，收集即時的監控數據。這種方式的優點包括：
- 由監控系統統一控管收集頻率與目標，方便管理與調整。
- 可以即時發現目標服務是否離線（因為拉不到資料）。
- 配合服務發現（如 Kubernetes Service Discovery）可自動追蹤動態變化的目標。

而有些其他監控系統或資料庫（如 ClickHouse、Graphite），則支援「推送（Push）」模型。這種模式下，由應用程式主動將監控數據透過 HTTP、UDP 等協定送到指定的 metrics server。這種方式的優點包括：
- 適合短生命週期的任務（如 batch job），可在任務結束時主動推送一次資料。
- 易於將資料同時推送到多個接收端。

### 具體例子
- **Prometheus**：預設為「拉取（Pull）」模型，主動定期抓取目標的 metrics endpoint。若需支援 push，可透過 Pushgateway 作為中介。
- **ClickHouse**：支援 Prometheus 的 remote-write（Push）協定，允許應用程式直接將 metrics 以 HTTP 方式推送進 ClickHouse。
- **Graphite**：原生支援「推送（Push）」模型，應用程式直接將資料送到 Graphite。

**總結**：
- **拉取（Pull）模型**：Prometheus、Kubernetes Metric Server
- **推送（Push）模型**：ClickHouse（remote-write）、Graphite
- 實務上，兩種模型各有優缺點，選擇時可依據應用場景與基礎設施需求彈性調整。
