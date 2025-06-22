# Kubernetes 中的各種 Probe

[English](../en/16_k8s_probes.md) | [繁體中文](../zh-tw/16_k8s_probes.md) | [日本語](../ja/16_k8s_probes.md) | [回到索引](../README.md)

在 Kubernetes 中，Probe 是由 Kubelet 對容器（Container）定期執行的健康檢查，以確保應用程式的可靠性。根據不同的檢查目的，Probe 主要分為三種類型。

### Probe 概覽

*   **`StartupProbe` (啟動探測)**: 判斷容器內的應用程式是否已經「啟動完成」。這對啟動緩慢的應用程式特別有用，可以防止它們在準備就緒前就被其他 Probe 錯誤地砍掉。
*   **`LivenessProbe` (存活探測)**: 判斷應用程式是否仍在「運行且健康」。如果探測失敗，Kubelet 會重新啟動容器，主要用於「監控自身」。
*   **`ReadinessProbe` (就緒探測)**: 判斷應用程式是否「準備好接收流量」。如果探測失敗，Kubernetes 會暫停向其發送請求。其主要用於「監控外部依賴」，如資料庫連線、其他 API 等。

### Probe 流程圖

這張圖描繪了從 Pod 啟動到各種 Probe 開始運作的完整生命週期：

```text
+-----------------+
| Pod             |
| Initialization  |
+-----------------+
        |
        v
+------------------------------------------------------------------+
| StartupProbe                                                     |
| (檢查應用程式是否已啟動)                                             |
|                                                                  |
| [成功] ---------------------> 啟動 LivenessProbe & ReadinessProbe |
| [失敗] ---------------------> [重新啟動 Pod]                       |
+------------------------------------------------------------------+
        |
        | (StartupProbe 成功後)
        |
        +----------------------------------------------------------+
        |                                                          |
        v                                                          v
+----------------------------+                     +-----------------------------+
| LivenessProbe              |                     | ReadinessProbe              |
| (監控應用程式自身是否僵化)     |                     | (監控外部依賴是否就緒)          |
|                            |                     |                             |
| [失敗] -> [重新啟動 Pod]     |                     | [失敗] -> [標記為 Unready]    |
|                            |                     |              (停止接收流量)   |
| [成功] -> (持續監控)         |                     |              |              |
|                            |                     |              v              |
|                            |                     | [成功] -> [標記為 Ready]      |
|                            |                     |              (恢復接收流量)   |
+----------------------------+                     +-----------------------------+
```

### 各 Probe 的特性

| 特性 | StartupProbe (啟動探測) | LivenessProbe (存活探測) | ReadinessProbe (就緒探測) |
| :--- | :--- | :--- | :--- |
| **用途** | 主要用於需要較長啟動時間的應用，防止在啟動完成前被 `LivenessProbe` 意外終止。 | 確保應用程式處於運行狀態，常用於解決死鎖等問題。 | 判斷應用程式是否「準備好」接收流量，常用於服務啟動時的依賴檢查。 |
| **監控對象** | 應用程式自身 | 應用程式自身 | 應用程式的外部依賴<br/>(如：DB 連線、其他 API) |
| **失敗行為** | Kubelet 重新啟動 Pod | Kubelet 重新啟動容器 | Endpoint Controller 將 Pod 從 Service 中移除，暫停流量。 |
| **成功行為** | 不再執行，並交由後續 Probe 接手。 | 容器持續運行，並定時監控。 | Pod 被加回 Service 中，開始（或恢復）接收流量。 |

### 建議與配置

在設計 Probe 時，可以遵循以下幾個最佳實踐：

#### 何時使用哪種 Probe？

1.  **`StartupProbe`**: 當你的應用程式啟動時間很長，超過 `LivenessProbe` 的 `failureThreshold * periodSeconds` 時，就應該使用 `StartupProbe`。
2.  **`LivenessProbe`**: 幾乎所有長時間運行的服務都應該配置。它可以捕捉到應用程式崩潰或死鎖等問題。但探測邏輯應盡可能輕量，避免對應用程式造成額外負擔。
3.  **`ReadinessProbe`**: 當你的應用程式需要依賴外部系統（例如資料庫、快取、其他 API），或者在啟動後需要執行一些初始化工作才能提供服務時，就必須使用 `ReadinessProbe`。這能確保只有在完全就緒時，流量才會進來，實現優雅的服務啟動 (graceful start) 與滾動更新 (rolling updates)。

#### Probe Endpoint 設計的差異

一個常見的錯誤是讓 Liveness Probe 和 Readiness Probe 使用同一個 health check endpoint。這是不好的實踐，因為它們的關注點不同：

*   **Liveness Endpoint (`/healthz`, `/livez`)**: 應該只回報應用程式本身的狀態，不應檢查外部依賴。如果因為資料庫暫時不可用而導致 Liveness Probe 失敗，Pod 會被不必要地重啟，可能引發連鎖反應的重啟風暴。
*   **Readiness Endpoint (`/readyz`, `/ready`)**: 應該檢查應用程式提供完整服務所需的一切，包括與後端資料庫或其它服務的連線。當後端服務出問題時，Readiness Probe 失敗會讓 Pod 暫時離線，待後端服務恢復後再自動上線，這是更優雅的處理方式。

#### 配置範例

這是一個結合了三種 Probe 的 Pod 配置範例：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: my-app-container
    image: my-app:1.0
    ports:
    - containerPort: 8080
    
    # StartupProbe: 給予應用程式最多 5 分鐘 (30 * 10s) 的啟動時間
    startupProbe:
      httpGet:
        path: /api/health/startup
        port: 8080
      failureThreshold: 30
      periodSeconds: 10

    # LivenessProbe: 啟動後，每 15 秒檢查一次是否存活
    livenessProbe:
      httpGet:
        path: /api/health/liveness
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 15
      timeoutSeconds: 2
      failureThreshold: 3

    # ReadinessProbe: 啟動後，每 20 秒檢查一次是否能提供服務
    readinessProbe:
      httpGet:
        path: /api/health/readiness
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 20
      timeoutSeconds: 5
      failureThreshold: 3
```


