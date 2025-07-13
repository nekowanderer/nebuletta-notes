# AWS ECS：EC2 Mode 與 Fargate Mode 差異

[English](../en/18_aws_ecs.md) | [繁體中文](../zh-tw/18_aws_ecs.md) | [日本語](../ja/18_aws_ecs.md) | [回到索引](../README.md)

## ECS 基本架構說明
- ECS (Elastic Container Service) 是 AWS 提供的容器管理平台。
- 基本概念：你把想執行的應用包裝成 Docker Image，交給 ECS，ECS 幫你在背後的運算資源（EC2 或 Fargate）啟動、管理、監控這些容器。

- EC2 Mode:
```
                                                                                        +-----------------------------------------------------------------+ 
                                                                                        | +-----------------+     +-------------------------------------+ | 
                                                                                        | |    Service      |---->|           Target Status             | | 
                                                                                        | +-----------------+     +-------------------------------------+ | 
                                                                                        | +-----------------+     +-------------------------------------+ | 
                                                                                        | |    Service      |---->|           Target Status             | | 
                                                                                        | +-----------------+     +-------------------------------------+ | 
                                                                                        |                                                                 | 
                                                                                        | +-----------------+     +-----------Target Status-------------+ | 
                                     |  +---------+                                     | |     Service     |---->| +--------+  +--------+   +--------+ | | 
                                     |  | Cluster |------------------------------------>| +-----------------+     | | Task01 |  | Task02 |   | Task03 | | | 
                                     |  | Manager |                                     |  |-Task Number          | | (slot) |  | (slot) |   | (slot) | | | 
                                     |  +---------+                                     |  |                      | +--------+  +--------+   +--------+ | | 
                                     |     ^   ^                                        |  +-Task Definition      +----|-----------|------------|-------+ | 
                                     |     |   |                                        |    |                         |           |            |         | 
                                     |     |   |                                        |    +-Container Definistion   |           |            |         | 
                                     |     |   |                                        |      |                       |           |            |         | 
                                     |     |   |                                        |      +-Image                 |           |            |         | 
                                     |     |   |                                        |        |                     |           |            |         | 
                                     |     |   |                                        |        +-Startup Command     |           |            |         | 
                                     |     |   |                                        |                              |           |            |         | 
                                     |     |   |                                        +------------------------------|-----------|------------|---------+ 
                                     |     |   |   +-------------------------------+                                   |           |            |           
                                     |     |   |   | +-----------+ +-------------+ |                                   |           |            |           
                                     |     |   +---| | Container | | Container04 |<------------------------------------+           |            |           
  +------------------------------+   |     |       | | Instance  | +-------------+ |                                               |            |           
  | +--------+ +---------------+ |   |     |       | +-----------+                 |                                               |            |           
  | | Docker | | ECS Container | |   |     |       +-------------------------------+                                               |            |           
  | | Engine | | Agent         | |   |     |       +-------------------------------+                                               |            |           
  | +--------+ +---------------+ |   |     |       | +-----------+ +-------------+ |                                               |            |           
  | +--------------------------+ |   |     +-------| | Container | | Container02 |<------------------------------------------------+            |           
  | |          Linux           | |   |             | | Instance  | +-------------+ |                                                            |           
  | +--------------------------+ |   |             | +-----------+ +-------------+ |                                                            |           
  +------------------------------+   |             |               | Container03 |<-------------------------------------------------------------+           
                                     |             |               +-------------+ |                                                                        
                                     |             +-------------------------------+                                                                        
                                     |              +-----+ +--------------------+                                                                          
                                     |              | ELB | | Auto Scaling Group |                                                                          
                                     |              +-----+ +--------------------+                                                                                                                                                                    
```

- Fargate Mode:
```
                                +-----------------------------------------------------------------+                                                   
                                | +-----------------+     +-------------------------------------+ |                                                   
                                | |    Service      |---->|           Target Status             | |                                                   
                                | +-----------------+     +-------------------------------------+ |                                                   
                                | +-----------------+     +-------------------------------------+ |                                                   
                                | |    Service      |---->|           Target Status             | |                                                   
                                | +-----------------+     +-------------------------------------+ |                                                   
                                |                                                                 |                                                   
                                | +-----------------+     +-----------Target Status-------------+ |                                                   
+---------+                     | |     Service     |---->| +--------+  +--------+   +--------+ | |                                                   
| Cluster |-------------------->| +-----------------+     | | Task01 |  | Task02 |   | Task03 | | |                                                   
| Manager |                     |  |-Task Number          | | (slot) |  | (slot) |   | (slot) | | |                                                   
+---------+                     |  |                      | +--------+  +--------+   +--------+ | |                                                   
   ^   ^                        |  +-Task Definition      +----|-----------|------------|-------+ |                                                   
   |   |                        |    |                         |           |            |         |                                                                                                   
   |   |                        |    +-Container Definistion   |           |            |         |                                                   
   |   |                        |      |                       |           |            |         |                                                   
   |   |                        |      +-Image                 |           |            |         |                                                   
   |   |                        |        |                     |           |            |         |                                                   
   |   |                        |        +-Startup Command     |           |            |         |                                                   
   |   |                        |                              |           |            |         |                                                   
   |   |                        +------------------------------|-----------|------------|---------+                                                   
   |   |                                                       |           |            |                                                             
   |   |     +--------------+                                  |           |            |                                                             
   |   +-----|              |<---------------------------------+           |            |                                                             
   |         |              |                                              |            |                                                             
   |         |              |                                              |            |                                                             
   |         |   Fargate    |<---------------------------------------------+            |                                                             
   |         |              |                                                           |                                                             
   |         |              |                                                           |                                                             
   +---------|              |<----------------------------------------------------------+                                                             
             +--------------+                                                                                                                         
```

## ECS EC2 Mode 與 Fargate Mode 差異總覽
| 特性       | EC2 Mode                         | Fargate Mode             |
| -------- | -------------------------------- | ------------------------ |
| 運算資源管理   | 你自己建立/管理 EC2 Instance            | AWS 自動管理，開發者無需碰 VM       |
| 自行管理的彈性  | 需設定 Auto Scaling Group、ELB       | 自動彈性調整，不需管理底層資源          |
| IAM 權限細節 | 可控制 EC2/Container Instance 的 IAM | 只需關注 Task、Service 的 IAM  |
| 運算資源擴展   | 需管理 EC2 數量，手動/自動擴展               | 只需定義 Task 數量，Fargate自動解決 |
| 成本結構     | 以 EC2 Instance 為單位計費             | 以 Task 所需 vCPU/Memory 計費 |
| 適用場景     | 有特殊網路需求、需細控基礎設施                  | 想省事、無須管理基礎設施             |


## Container Instance 結構解析（EC2 Mode）
| 元件                  | 說明                                 |
| ------------------- | ---------------------------------- |
| Linux               | 底層 OS，通常是 Amazon Linux 2/Ubuntu    |
| Docker Engine       | 容器運行環境，負責啟動/停止容器                   |
| ECS Container Agent | 負責與 ECS Cluster Manager 溝通，管理本機上容器 |
- ECS Container Agent 是 AWS 提供的 Daemon 程式，預設已安裝在 AWS 官方 ECS AMI 上。
- 它的主要職責：讓 EC2 能接收 ECS Cluster Manager 的指令（像是哪個 Task 要啟動、狀態回報等）。

## IAM Role 種類與差異
| Role 類型                | 授權範圍                     | 官方用途說明                                       |
| ----------------------- | ------------------------ | -------------------------------------------- |
| Service Role            |  ECS Service              | 提供 Service 操作 AWS 資源（如 Auto Scaling, ELB）    |
| Task Execution Role     |  ECS Task Definition      | 讓 ECS 執行容器時可抓 ECR 映像、寫 CloudWatch Logs       |
| Task Role               |  執行中的 Container/Task      | 給 Task (應用程式) 需要用到的 AWS 資源存取權限（S3、DynamoDB等） |
| Container Instance Role |  Container Instance (EC2) | 給 ECS Agent/EC2 本身用於與 ECS 管理溝通等              |

#### 官方說明補充：
- Task Role 是直接 assign 給應用程式容器用的 AWS 權限，只會被該 Task 內的容器拿來用（如存取 S3、DynamoDB）。
- Task Execution Role 是 ECS Agent 幫你拉 image、上傳 log、取得 secret 等時用的，通常只需要最小權限。
- Service Role 是少數情境會用到（如 Service Auto Scaling），大部分情境不需特別設定。
- Container Instance Role 只在 EC2 Mode 用到，Fargate Mode 下不需設定。

## Cluster Manager 與 Task 目標狀態
- Cluster Manager 是 ECS 的大腦，負責「指派」各種 Service、Task 到對應的容器節點上運行。
- Task 目標狀態：例如你設定 Service 需要 100 個 Task，Cluster Manager 不在乎實際是哪些 EC2 Instance、Fargate Node 執行，只關心目前活著的 Task 數量是否達標。
  - 如果 Task 掛掉，Cluster Manager 會馬上補新 Task 確保目標狀態一致。
  - 實際運作時，Task 會被分配到不同 Instance（EC2 或 Fargate node），但這對你透明。

## ELB 與 Auto Scaling Group 的角色
| 元件                          | 主要作用                                                     |
| --------------------------- | -------------------------------------------------------- |
| ELB (Elastic Load Balancer) | 幫容器服務分流外部流量，對外提供單一 Endpoint                              |
| Auto Scaling Group          | 讓 EC2 Container Instance 可根據 Task 數量自動增減，動態配合流量與 Task 負載 |

#### 在雲端環境的優勢：
- 若需要 10 萬個 Task，Auto Scaling Group 可根據你設定的 scaling policy 自動增減 EC2 Instance，並自動把新機器註冊到 ECS Cluster。
- 在 Fargate 下更方便，你只要調高 Task 數量，AWS 會自動準備足夠的運算資源，不用考慮 Instance 管理與 Scaling。

## 官方觀點與常見誤區補充
- Task Role 的指派： 每個 ECS Task 可以擁有不同的 Task Role，這點是 Fargate/EC2 都支援的，也是 AWS 強調的最細緻權限模型（Least Privilege）。
- Fargate 無需管理 Instance: 在 Fargate 模式下，不用設定 Container Instance Role，權限都由 Task Role/Task Execution Role 決定，底層 Instance AWS 自行管理，你看不到也不能自訂。
- 彈性擴展： Fargate 由 AWS 負責資源調度，EC2 則需考慮 ASG scaling 和 Instance 壽命管理。

## 關於 Fargate Mode 之下的 Auto Scaling

### Fargate 實作底層到底有沒有 ASG？
- 對使用者來說：在 Fargate Mode 下，你完全看不到也無法直接管理 Auto Scaling Group (ASG) 或 EC2 Instance。
- AWS 背後實作：AWS 確實在自己內部會有資源池（底層仍然有 VM/ASG 做調度），但這部分完全由 AWS 管理、最佳化，開發者不可見也無法設定。
- 重點：你在 AWS Console、CLI、API 都看不到任何與 ASG 相關的設定或監控頁籤。ASG 僅存在於 AWS 的「黑盒子」維運流程，保證有足夠資源執行你的 Task。

### Fargate 用戶還能設定哪些 scaling 參數？
- Service scaling：你可以設定 ECS Service 的目標 Task 數量，以及根據指標（如 CPU/Memory 使用率、Queue 長度等）做自動擴縮（Auto Scaling）。
  - 這是對「Task 數量」進行調整，不是調整 VM 數量。
  - ECS 會根據 scaling policy 自動啟動/停止 Task，Fargate 會自動準備/釋放底層運算資源。
- Scaling 敏感度（例如：
  - 你可以用 Application Auto Scaling 設定 scaling policy，例如
    - 當平均 CPU 超過 70% 持續 5 分鐘，就加 N 個 Task
    - 當 SQS queue 長度超過 100，就加 Task
  - 這些都是設定在 ECS Service 層級，不是直接設定底層 VM/ASG。

### 在 Fargate 下「還能」設定的細節
| 可調項目                      | 是否可設定 | 範例                                           |
| ------------------------- | ----- | -------------------------------------------- |
| 每個 Task 的 vCPU/Memory    | ✅     | 0.25 vCPU / 0.5GB \~ 16 vCPU / 120GB         |
| Task 數量                   | ✅     | 例如：Service 最小 1 最大 500                       |
| Service scaling policy     | ✅     | Application Auto Scaling, Alarm-based policy |
| IAM Task Role              | ✅     | 每個 Task 可指定不同 IAM Role                       |
| VPC/Subnet/Security Group  | ✅     | 可以選擇 Fargate 任務要跑在哪些 VPC/Subnet 上            |
| EC2 Instance 類型           | ❌     |  |
| ASG scaling policy         | ❌     |  |
| Instance IAM Role          | ❌     |  |
| EC2 instance 的 Life Cycle | ❌     |  |

### 結論
- Fargate 下沒有「可見的」ASG，所有彈性調度、底層機器由 AWS 黑盒自動處理。
- 你唯一能調的彈性規則，就是任務（Task）或服務（Service）層級的 Auto Scaling。
- 你仍然可以決定scaling 敏感度、Trigger、以及最大/最小 task 數，但不能直接管理 VM 或 ASG。

## 重點總結
- EC2 Mode 適用於有特殊資源、網路需求或希望節省成本時
  - 需自行管理 Instance、權限與網路設定
- Fargate Mode 適用於無基礎設施管理需求，完全按需自動彈性運算
  - 只需設定 Task 與 Service，其他 AWS 幫你打理

