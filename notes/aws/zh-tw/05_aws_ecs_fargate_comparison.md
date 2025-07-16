# ECS、Fargate、Lambda、EKS 比較與 Sidecar 設計說明

[English](../en/05_aws_ecs_fargate_comparison.md) | [繁體中文](05_aws_ecs_fargate_comparison.md) | [日本語](../ja/05_aws_ecs_fargate_comparison.md) | [回到索引](../README.md)

## EC2 vs Fargate

| 面向 | EC2 | Fargate |
|------|-----|---------|
| 本質 | VM（虛擬機） | Serverless 容器運行環境 |
| 控制權 | 自行管理 VM（更新、監控、修補） | AWS 管理基礎設施，只需負責容器設定 |
| 成本結構 | 長時間開機可能浪費錢 | 只為實際使用時間付費，更節省 |
| 運維責任 | 需要 patch OS、調整資源等 | AWS 自動管理，開發者專注應用邏輯 |
| 適合場景 | 長時間運行、有特殊需求 | 任務型工作、自動彈性擴展需求 |

### 使用情境
- EC2 適合：
  - 需要完整控制環境
  - 有需要使用特殊 kernel module 的應用
  - 有長時間運行、或不容易 containerize 的程式
  - 相當於天堂裡面召喚出來的食人妖精
- Fargate 適合：
  - 短時任務、事件觸發的運算
  - 快速啟動、快速銷毀（像是批次處理）
  - 不想管主機、只想交付應用的團隊
  - 沒有實體的召喚單元 (?)

---

## ECS on EC2 vs ECS on Fargate

| 比喻 | ECS on EC2 | ECS on Fargate |
|----------|----------------------------------|--------------------------------|
| 召喚獸性格 | 「你叫他來、還要幫他開房間和準備食物」 | 「你叫他來，他自己找房、自己帶便當」 |
| 資源管理 | 你自己管理 EC2 instance（CPU、RAM、Auto Scaling） | AWS 自動配置所需資源，無需管理 EC2 |
| 成本 | 召了肥肥就要養（無論有沒有跑任務） | 只為執行任務時段付費（更像 pay-per-use） |
| 擴展性 | 彈性需設 ASG 或手動調整 | 原生彈性，高度自動化 |
| 運維責任 | EC2 patching、資源壓力、容量預估等都你來 | AWS 處理 infra，開發者專注應用程式邏輯 |
| 適用情境 | 成本控制明確、環境特殊需求、運行時間長 | 小團隊、快速開發、需求變動大的情境 |

---

## Fargate vs Lambda

| 面向 | Fargate | Lambda |
|------|---------|--------|
| 召喚獸性格 | 「會做一整個任務、可以接長工」 | 「做一次事情就跑，快狠準的刺客」 |
| 啟動方式 | 可以長時間運作的容器任務（背景作業、API、批次） | 通常為事件觸發（API Gateway、S3、CloudWatch 等） |
| 維持時間 | 幾分鐘～幾小時都可以 | 最長 15 分鐘（預設冷啟動） |
| 控制彈性 | Docker-based，支援多種語言與 runtime | 支援有限語言，runtime 要 AWS 支援 |
| 運行成本 | 計算 CPU 與記憶體（依配置與時間） | 依函式運行時間與記憶體計費（ms 為單位） |
| 適合場景 | 任務有狀態、或需要更多 runtime 控制 | 任務短小精幹、如轉檔、觸發通知、image resize |

---

## Auto Scaling Group vs ECS Service Auto Scaling

| 名稱 | 管的是誰 | 適用在哪裡 | 調整對象 |
|------|----------|-------------|-----------|
| Auto Scaling Group (ASG) | EC2 數量 | ECS on EC2 | VM 數量 |
| ECS Service Auto Scaling | Task 數量 | ECS on Fargate / EC2 | Container 任務數量 |

### 更深入說明：
#### Auto Scaling Group（ASG）
- 你在 EC2（或 ECS on EC2）環境下使用。
- 調整的是 虛擬機的數量。
- 例如你設了 ASG 的 min=2、max=10，然後 CPU 壓力大於 70%，就會自己新增一台 EC2。

#### 適用場景：
- EC2 為主、VM 需要擴充時。
- 在 ECS on EC2 中，你的 container 跑不下了，才需要用 ASG 增加 EC2 來跑更多 task。

#### ECS Service Auto Scaling（適用 Fargate & EC2）
- 調整的是 ECS task 數量，也就是 container 的 instance 數量。
- 你可以設定 Target Tracking，比如讓平均 CPU 保持在 50%，他就會自己增加 task。
- 如果是 Fargate，新增的 task 就直接由 AWS 在背後幫你找空間執行。
- 如果是 EC2，你得先確認 EC2 VM 還有空間跑，不然會排在 pending。

#### 適用場景：
- 你想讓容器的數量自動上下調整。
- 在 Fargate 上完全自動化，不需要先顧底層 VM。
- 在 EC2 上使用，要配合 ASG 才能達到完整彈性。

---
## ECS Task ≠ Container Instance

- Q: 所以，ECS task 等價於 container instance 嗎?
- A: 這句話要小心解讀，因為 ECS task ≠ container instance，雖然它們在某些上下文中看起來很像，但其實意思不一樣。

| 名詞                                | 說明                                                                        | 比喻                                       |
| --------------------------------- | ------------------------------------------------------------------------- | ---------------------------------------- |
| **ECS Task Definition**           | 是容器組的定義，必須在此定義好 task 所需的各種條件，譬如運算資源需求以及 container image | 背包的製造藍圖，也可以說是 K8S 當中的 Pod template |
| **ECS Task**                      | 是「一組容器」的運行實例，根據 task definition 執行。<br>可以只跑一個 container，也可以是多個 container。 | 「任務包」：裝著 1 個或多個 container 的小背包，接到任務就出發，也可以想成是 K8S 中的 Pod |
| **ECS Service** | 管理 service 的數量與 load balancing 設定，把流量導向後面的 ECS Task | 相當於 K8S Service + K8S Deployment |
| **Container Instance**            | 這個詞**只在 ECS on EC2 才會出現**，意思是被註冊到 ECS 的一台 EC2。                            | 「容器宿主主機」：一棟可以放很多 ECS task 的房子          |
| **Fargate 沒有 Container Instance** | Fargate 是 serverless，沒有 EC2。Task 是直接被 AWS 放到某個他幫你管的地方跑。                   | 雲中小套房：AWS 自動分配給每個任務一間獨立房間，你不用管地址或清潔費     |

#### 差異
| 問題                      | Fargate                     | ECS on EC2                       |
| ----------------------- | --------------------------- | -------------------------------- |
| 有沒有 Container Instance？ | ❌ 沒有                        | ✅ 有                              |
| 一個 Task 跑幾個 Container？  | 看你 task definition，通常 1～N 個 | 一樣                               |
| 誰幫你找地方跑 Task？           | AWS 自動                      | ECS 排程器從 container instance 裡找空位 |


---

## Sidecar Pattern 實例：Microservice + OTEL + Fluent Bit

#### 為什麼一個 ECS Task 可能包含多個 container？
因為：
> 一個 ECS Task 是根據 Task Definition 啟動的，這個定義裡可以包含多個 container。

這種設計就是用來支援 sidecar pattern，讓不同的 container 共用：
  - 相同的 網路命名空間（Network namespace）
  - 相同的 儲存空間（Volumes）
  - 並一起被調度、一起被管理

| Container          | 角色定位                   | 功能說明                                                                 |
| ------------------ | ---------------------- | -------------------------------------------------------------------- |
| 🧩 microservice | 主容器                    | 實際提供業務邏輯的 API 或服務                                                    |
| 📈 OTEL Collector  | Metrics/Traces sidecar | 收集、轉送 OpenTelemetry 格式的 metrics & tracing（通常是 OTLP over gRPC 或 HTTP） |
| 📤 Fluent Bit      | Logs sidecar           | 收集 stdout logs，並轉送到如 CloudWatch、S3、Elasticsearch 等 log sink          |

這種架構實現了 logs、metrics、traces 三大觀測性管道分離，但部署在同一 Task 裡，好處如下：
- 解耦 observability 流程與主程式碼：
  - 不需要在 microservice 裡硬塞 log/trace/export 的邏輯。
  - OTEL Collector 和 Fluent Bit 都可以由 infra team 維護。
- 可重用性高：
  - 多個 microservice 都可以用同樣的 sidecar pattern。
  - 只要微調 config file 即可支援不同的需求。
- 低耦合、高彈性部署：
  - 假設有一天想把 OTEL Collector 拆出來變成 side service（而非 sidecar），也容易遷移。
- 支援共享 volume、port 通訊、dependsOn 設定，形成獨立 task 部署單位。

#### sidecar container 的常見用途

| 類型                    | 功能                                                     |
| --------------------- | ------------------------------------------------------ |
| Logging               | 傳送 log 到 S3、CloudWatch、Elasticsearch                   |
| Metrics               | Prometheus exporter、Datadog agent                      |
| Proxy                 | Envoy, NGINX, Linkerd 等作為 reverse proxy / service mesh |
| Security              | 如 Istio Citadel 提供憑證、自動更新憑證的 agent                     |
| Init container（非 ECS） | 這在 K8s 才有，ECS 沒有這個機制                                   |

---

## ECS 不支援 initContainer 與 preStop

ECS 的 task 是根據 task definition 一起啟動 container 的，但它：

| 功能                                  | ECS 支援？    |
| ----------------------------------- | ---------- |
| 指定 container 啟動順序（`dependsOn`）      | ✅ 有基本支援    |
| init container（只跑一次的初始化容器）          | ❌ 沒有       |
| preStop hook / graceful shutdown    | ❌ 沒有內建支援   |
| sidecar restart policy 分離控制         | ❌ 沒有       |
| container healthcheck 掛掉要不要殺整個 task | ✅ 可設定，但較粗糙 |

所以 ECS 的設計就是：
>「所有 container 綁成一包 task，要生要死一起來。」

不像 K8s 可以細緻地定義：「某個 container 是 init 只跑一次、某個 container 掛了不影響主容器、某個要等別人 ready 才啟動」。

---

## 如需細緻的 Lifecycle 控制，建議使用 AWS EKS

#### 什麼時候該用 EKS
| 你有這些需求時                          | 建議使用 EKS |
| -------------------------------- | -------- |
| 容器之間複雜的相依關係                      | ✅        |
| 多階段初始化流程                         | ✅        |
| 需要強化的可觀測性、監控、或網路策略               | ✅        |
| 想整合 Istio、Linkerd 等 Service Mesh | ✅        |
| 想自訂容器 lifecycle hook 行為          | ✅        |

#### Lifecycle
| 特性 | EKS 支援 |
|------|----------|
| initContainer | ✅ |
| preStop hook | ✅ |
| Readiness / Liveness probe | ✅ |
| terminationGracePeriod | ✅ |
| Service Mesh / Sidecar decoupling | ✅ |

若需要嚴謹的容器生命週期控制與彈性架構建議使用 AWS EKS（Kubernetes 託管服務）。

## ECS v.s. EKS

| 面向     | ECS                | EKS               |
| ------ | ------------------ | ----------------- |
| 操作難度   | 簡單（可視為 "K8s-lite"） | 較複雜，要學 Kubernetes |
| 成本     | 可控、資源使用更單純         | 管理成本高，可能多付錢買複雜性   |
| 擴充彈性   | 中等                 | 高，適合大型系統          |
| 社群與工具鏈 | AWS 生態為主           | 全世界的 K8s 生態支援你    |
