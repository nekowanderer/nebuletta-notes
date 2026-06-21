# SQS 連線池分析與配置

[English](../en/10_sqs_connection_pool_analysis_and_configuration.md) | [繁體中文](10_sqs_connection_pool_analysis_and_configuration.md) | [日本語](../ja/10_sqs_connection_pool_analysis_and_configuration.md) | [返回索引](../README.md)

---

## 背景
- 實驗日期: 2025/12/09
- 難度：🤬🤬🤬
- 描述: Production 環境中，新增第四個 SQS queue 後，出現連線池耗盡錯誤。錯誤訊息顯示無法在設定的最大時間內從連線池取得連線。

## 遇到的現象

- Kibana log：
  ```
  Unable to execute HTTP request: Acquire operation took longer than the configured maximum time.
  This indicates that a request cannot get a connection from the pool within the specified maximum time.
  ```
- 錯誤發生在 `QueueAsyncClientImpl` 處理 non-transactional EMAIL messages 時 (底層直接呼叫 AWS SDK V2)
- 服務無法正常處理來自新 queue 的訊息

## 除錯過程

### 第一階段：確認基礎設施設定

已確認以下各項目都設定正確：

| 項目 | 狀態 | 說明 |
|------|------|------|
| SQS Queues | ✅ | 原本有 3 個 queue，新增第四個 queue |
| Poller 設定 | ✅ | 每 1 秒拉一次訊息 (`sqs.communication.poller.schedule: "1s"`), 這個是程式自訂的 property |
| Batch Size | ✅ | Transactional: 10, Non-transactional: 10, 就是每次都從 SQS 拿幾筆 message, max = 10 |
| SQS Client | ⚠️ | 使用 AWS SDK 預設配置 (問題就出在這) |

### 第二階段：分析 Queue 運作模式

檢查各個 queue 的運作方式：

**原本的 3 個 Queue：**
- `transactionalQueueUrl`: Transactional message queue
- `queueUrl`: Non-transactional SMS message queue  
- `emailMetricsQueueUrl`: Email metrics queue

**新增的第四個 Queue：**
- Non-transactional EMAIL message queue
- 每次 poller 執行：
  - 1 次 `receiveMessage` 呼叫（批次讀取最多 10 個訊息）
  - 10 次 `deleteMessage` 呼叫（每個訊息一次）
  - **總共需要 11 個 HTTP 連線**

**結論**：新 queue 的運作模式是連線密集型的，每次執行需要 11 個連線。

### 第三階段：連線池容量分析

檢查 SQS Client 的連線池配置：

| 配置項目 | 預設值 | 實際需求 |
|---------|--------|----------|
| `max-concurrency` | 50 | 不足 |
| `connection-acquisition-timeout-seconds` | 2 秒 | 太短 |
| `max-pending-connection-acquires` | 10,000 | 足夠 |

**發現**：
- 預設連線池只有 50 個連線
- 新 queue 每秒需要 11 個連線，加上其他 queue 的流量，很容易超過 50 個連線
- 連線取得超時時間只有 2 秒，在高流量情況下不夠等待連線釋放

## 問題的根本原因

### 新增 Queue 導致連線池耗盡

**關鍵發現**：新增第四個 queue 時，該 queue 採用連線密集型運作模式（每次執行需要 1 個 receiveMessage + 10 個 deleteMessage = 11 個連線），大幅增加了並發 HTTP 連線需求，超過了預設連線池容量 50。

### 架構變更

**變更前（3 個 Queue）：**
- `transactionalQueueUrl`: Transactional message queue
- `queueUrl`: Non-transactional SMS message queue  
- `emailMetricsQueueUrl`: Email metrics queue
- 每個 poller 執行處理這些 queue 的訊息

**變更後（4 個 Queue - 新增後）：**
- `transactionalQueueUrl`: Transactional message queue
- `queueUrl`: Non-transactional SMS message queue  
- `emailMetricsQueueUrl`: Email metrics queue
- **新 queue**: Non-transactional EMAIL message queue（新增）
  - 每個 poller 執行：
    - 1 次 `receiveMessage` 呼叫（批次讀取最多 10 個訊息）
    - 10 次 `deleteMessage` 呼叫（每個訊息一次）
    - **總共需要 11 個 HTTP 連線**

### 影響

- **連線密集型模式**：新 queue 每次 poller 執行需要 11 個連線（1 個 receiveMessage + 10 個 deleteMessage）
- **連線池壓力增加**：poller 每 1 秒執行一次，這會增加大量並發連線需求
- **連線池耗盡**：預設連線池（50 個連線）在結合其他 queue 的流量時不足
- **連線取得超時**：當連線池耗盡時，新的 API 呼叫無法及時取得連線

## 問題原理解析

### 連線池運作機制

1. **連線池位置**：
   - 連線池**不在 SQS 服務端**，而是在**應用程式端**（應用程式內部）
   - 由 `NettyNioAsyncHttpClient` 管理，因為我們用的是 Quarkus framework
   - 位於應用程式記憶體中
   - 用於重複使用 HTTP 連線

2. **連線池運作流程**：
   ```java
   // 當你呼叫
   sqsAsyncClient.receiveMessage(...)
   // SDK 內部會：
   // 1. 從連線池取得一個 HTTP 連線
   // 2. 使用這個連線發送 HTTP 請求到 SQS
   // 3. 接收回應
   // 4. 將連線歸還到連線池（供下次使用）
   ```

3. **為什麼需要連線池？**
   - 每個 SQS API 呼叫都需要建立 HTTP 連線
   - 連線建立有開銷（TCP 握手、TLS 握手）
   - 連線池可以重複使用連線，提升效能

### 連線池耗盡的原因

- 當所有連線都在使用中（例如處理其他 API 呼叫）
- 新的 receiveMessage 呼叫需要等待連線釋放
- 如果等待時間超過 `connectionAcquisitionTimeout`（預設 2 秒），就會發生超時

### 新 Queue 的連線需求

**正常流量分析：**

| Queue | receiveMessage | deleteMessage | 總連線數/執行 |
|-------|----------------|---------------|---------------|
| Queue 1 - Transactional | 1 次 | 最多 10 次 | 最多 11 個 |
| Queue 2 - Non-transactional SMS | 1 次 | 最多 10 次 | 最多 11 個 |
| Queue 3 - Email metrics | 依實際使用 | 依實際使用 | 依實際使用 |
| **Queue 4 - 新 EMAIL queue** | **1 次** | **10 次** | **11 個** |

**連線池需求：**
- **新增新 queue 前**：現有 3 個 queue 及其各自的流量模式
- **新增新 queue 後**：新 queue 每秒額外增加 11 個連線
- **總並發連線數**：在正常運作時很容易超過 50（預設連線池大小）
- **突發情況**：連線需求可能會大幅增加

**突發流量估算：**
- 突發流量可能是正常流量的 10 倍
- 僅新 queue 在突發時就可能產生 110 個連線/秒（11 個連線 × 10 倍）
- 結合其他 queue 的流量，總並發連線需求可能超過 200

## 解決方案

### 建立自訂 SQS Client Producer

建立自訂的 SQS client Producer (`AwsSqsClientProducer`)，配置適當的連線池參數。

### 生產環境配置

```yaml
aws:
  sqs:
    max-concurrency: 200                    # 處理 200 個並發請求
    max-pending-connection-acquires: 2000   # 允許最多 2000 個請求排隊
    connection-timeout-seconds: 3           # 連線建立超時 3 秒
    connection-acquisition-timeout-seconds: 10  # 連線取得超時 10 秒
    api-call-timeout-seconds: 15            # API 呼叫總超時 15 秒
    api-call-attempt-timeout-seconds: 15    # API 呼叫嘗試超時 15 秒
```

- 其中，`max-pending-connection-acquires` 沒有顯式指定的話會出 runtime exception，而經過計算後，覺得預設的 10,000 太高了，所以調低一點。

### 配置參數說明

| 參數 | 預設值 | 建議值 | 說明 |
|-----|--------|--------|------|
| max-concurrency | 50 | 200 | 最大並發請求數 |
| max-pending-connection-acquires | 10,000 | 2,000 | 最大排隊請求數 |
| connection-acquisition-timeout-seconds | 2 | 10 | 連線取得超時（秒） |
| connection-timeout-seconds | 10 | 3 | 連線建立超時（秒） |
| api-call-timeout-seconds | 無限制 | 15 | API 呼叫總超時（秒） |
| api-call-attempt-timeout-seconds | 無限制 | 15 | API 呼叫嘗試超時（秒） |

### 配置參數計算邏輯

**maxConcurrency (200)**：
- 用途：設定最大並發請求數
- 對於 HTTP/1.1：等同於最大連線數
- 對於 HTTP/2：連線數取決於每個連線允許的最大串流數
- 建議值：200（基於實際流量需求）

**maxPendingConnectionAcquires (2000)**：
- 用途：當達到 maxConcurrency 時，允許排隊的最大請求數
- 計算邏輯：
  - 正常情況：多個 queue 產生並發請求，包括新 queue 的 11 個連線/秒
  - 突發情況：可能輕易超過 200 個連線/秒
  - 如果處理時間是 0.5 秒，在突發時 0.5 秒內會有 100+ 個新請求
  - 如果所有 200 個連線都在使用中，最多會有 100+ 個請求需要排隊
  - 加上安全邊際：建議 `2000`
- 建議值：`2000`（可以處理突發情況而不會過度使用資源）

**connectionAcquisitionTimeout (10 秒)**：
- 用途：從連線池取得連線的超時時間
- 問題：在高流量情況下，2 秒可能不足以等待連線釋放
- 建議值：10 秒（給請求更多時間等待連線）

## 為什麼 QA 環境沒有問題？

### QA 環境配置差異

| 配置項目 | Production | QA |
|---------|---------|---------|
| Poller 頻率 | 1 秒 | 3 秒 |
| Transactional batch size | 10 | 10 |
| Non-transactional batch size | 10 | 5 |
| 新 queue 連線需求 | 11 個連線/秒 | 較低（由於較長的 poller 間隔） |
| 請求速率 | 較高（多個 queue + 新 queue） | 約生產環境的 1/3 |

### 分析

- QA 環境的請求速率明顯低於生產環境（約 1/3）
- 較長的 poller 間隔（3 秒）降低了每秒的連線需求
- 即使使用預設連線池（50 個連線），也足以處理 QA 流量
- 較長的 poller 間隔提供了更多時間在執行之間釋放連線
- 由於以上差異，導致 QA 環境的 load testing 沒有辦法偵測出此問題，且 production 的流量幾乎是 24 小時不間斷，就算是 non-peak hour 也會也不少資訊流入，所以才會導致用預設的設定上線後馬上出錯

## 關鍵學習重點

### 連線池位置的重要性

| 特性 | 說明 |
|------|------|
| 位置 | **應用程式端**，不在 SQS 服務端 |
| 管理 | 由 NettyNioAsyncHttpClient 管理 |
| 儲存 | 在應用程式記憶體中 |
| 用途 | 重複使用 HTTP 連線 |

### 連線池限制

連線池有上限，即 `maxConcurrency`：
- 連線池限制 = `maxConcurrency`（目前為 200）
- 當達到限制時，新請求會排隊等待
- 排隊限制 = `maxPendingConnectionAcquires`（目前為 2000）
- 等待時間限制 = `connectionAcquisitionTimeout`（目前為 10 秒）

### 架構圖解

```
Communication service (ECS Container)
├── AwsSqsClientProducer
│   └── SqsAsyncClient
│       └── NettyNioAsyncHttpClient  ← 連線池在這裡！
│           └── HTTP Connection Pool (maxConcurrency: 200)
│               ├── Connection 1 ──┐
│               ├── Connection 2 ──┤
│               ├── Connection 3 ──┤
│               └── ...            │
│                                  │
└── When calling SQS API           │
    ├── receiveMessage() ──────────┼──> 需要一個 HTTP 連線
    ├── deleteMessage() ───────────┼──> 需要一個 HTTP 連線
    └── sendMessage() ─────────────┼──> 需要一個 HTTP 連線
                                   │
                                   ▼
                           HTTP/HTTPS connection to AWS
                                   │
                                   ▼
                            SQS Service (AWS)
```

## 預防措施

1. **新增 Queue 時評估連線需求**
   - 計算每個 queue 的連線需求（receiveMessage + deleteMessage 數量）
   - 評估總並發連線需求是否超過現有連線池容量

2. **監控連線池使用情況**
   - 監控連線池使用率
   - 設定告警當連線池使用率超過閾值時

3. **預先調整連線池配置**
   - 在新增高流量 queue 前，先調整連線池容量
   - 考慮突發流量的需求

4. **測試環境與生產環境配置差異**
   - 了解測試環境與生產環境的配置差異
   - 在測試環境可能不會出現問題，但生產環境會，壓測也不見得能夠模擬出來

## 結論

- 這次的除錯經驗提醒我們，在 AWS SQS 應用中，連線池是一個容易被忽略但影響重大的元件。特別是當新增 queue 時，必須仔細評估每個 queue 的連線需求模式，才能確保服務能正常運作。

- Production 環境要經過仔細計算後套用客製化的設定，不要直接用預設值就上。

- **記住**：連線池在應用程式端，不在服務端。新增 queue 時，特別是採用連線密集型運作模式的 queue，必須評估連線池容量是否足夠！

## 給非技術人員的餐廳比喻

可以把 HTTP connection pool 想像成一間餐廳。

### 原本的狀況

- 餐廳原本只有 50 個座位。
- 每一個 SQS API request 就像一位想進餐廳吃飯的客人。
- 客人要先取得座位，才能開始用餐。
- 用餐結束後，座位就會釋放，提供下一位客人使用。

### 新增 Queue 後

新增第四個 queue，就像多開了一個新的接單入口。

因此：

- 同一時間來餐廳的客人變多。
- 原本 50 個座位很快就坐滿。
- 後來的客人只能在候位區排隊。
- 如果客人等待太久仍拿不到座位，就會放棄離開。

這個「等待太久後離開」，就相當於系統拋出：

```text
Acquire operation took longer than the configured maximum time.
```

---
## connectionAcquisitionTimeout 與 connectionTimeout 的差異

這兩個設定很容易混淆。

### connectionAcquisitionTimeout

代表： 
> 一個 request 最多願意等多久，來取得 connection pool 裡的一條可用 connection。

餐廳比喻： 
> 客人在門口最多願意等座位多久。

本次錯誤就是發生在這個階段。

### connectionTimeout

代表：
> 取得 connection 資源後，實際和遠端服務建立 TCP/TLS connection 時，最多允許等待多久。

餐廳比喻：
> 客人已經拿到座位後，服務人員最多花多久和廚房建立聯繫。

因此：

```text
connectionAcquisitionTimeout = 10 秒
connectionTimeout = 3 秒
```

不能混為一談。

---

## 參考資料
- [Configure the Netty-based HTTP client](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/http-configuration-netty.html) 
- [NettyNioAsyncHttpClient.Builder](https://sdk.amazonaws.com/java/api/latest/software/amazon/awssdk/http/nio/netty/NettyNioAsyncHttpClient.Builder.html)
