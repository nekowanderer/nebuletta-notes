# AWS ECS：EC2 Mode と Fargate Mode の違い

[English](../en/18_aws_ecs.md) | [繁體中文](../zh-tw/18_aws_ecs.md) | [日本語](../ja/18_aws_ecs.md) | [インデックスに戻る](../README.md)

## ECS 基本アーキテクチャの説明
- ECS (Elastic Container Service) は AWS が提供するコンテナ管理プラットフォームです。
- 基本概念：実行したいアプリケーションを Docker Image にパッケージし、ECS に渡すと、ECS がバックエンドのコンピューティングリソース（EC2 または Fargate）でこれらのコンテナを起動、管理、監視してくれます。

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

## ECS EC2 Mode と Fargate Mode の概要比較
| 特徴 | EC2 Mode | Fargate Mode |
| -------- | -------------------------------- | ------------------------ |
| コンピューティングリソース管理 | 自分で EC2 Instance を作成/管理 | AWS が自動管理、VM 処理不要 |
| 自己管理の柔軟性 | Auto Scaling Group、ELB の設定が必要 | 自動弾性調整、基盤リソース管理不要 |
| IAM 権限の詳細 | EC2/Container Instance の IAM を制御可能 | Task、Service の IAM のみ関心 |
| コンピューティングリソース拡張 | EC2 数量管理、手動/自動拡張が必要 | Task 数量のみ定義、Fargate が自動解決 |
| コスト構造 | EC2 Instance 単位で課金 | Task の必要 vCPU/Memory で課金 |
| 適用シナリオ | 特殊なネットワーク要件、詳細なインフラ制御 | 簡便性重視、インフラ管理不要 |


## Container Instance 構造解析（EC2 Mode）
| コンポーネント | 説明 |
| ------------------- | ---------------------------------- |
| Linux | 基盤 OS、通常は Amazon Linux 2/Ubuntu |
| Docker Engine | コンテナ実行環境、コンテナの起動/停止を担当 |
| ECS Container Agent | ECS Cluster Manager との通信、ローカルマシン上のコンテナ管理 |
- ECS Container Agent は AWS が提供する Daemon プログラムで、AWS 公式 ECS AMI にデフォルトでインストールされています。
- 主な責任：EC2 が ECS Cluster Manager からコマンド（どの Task を起動するか、ステータス報告など）を受信できるようにします。

## IAM Role の種類と違い
| Role タイプ | 認可範囲 | 公式用途説明 |
| ----------------------- | ------------------------ | -------------------------------------------- |
| Service Role | ECS Service | Service が AWS リソースにアクセス（Auto Scaling、ELB など） |
| Task Execution Role | ECS Task Definition | ECS がコンテナ実行時に ECR イメージ取得、CloudWatch Logs 書き込み |
| Task Role | 実行中の Container/Task | Task（アプリケーション）が必要とする AWS リソースアクセス権限（S3、DynamoDB など） |
| Container Instance Role | Container Instance (EC2) | ECS Agent/EC2 自体が ECS 管理との通信などに使用 |

#### 公式説明補足：
- Task Role はアプリケーションコンテナ用に直接割り当てられる AWS 権限で、その Task 内のコンテナのみが使用（S3、DynamoDB アクセスなど）。
- Task Execution Role は ECS Agent がイメージプル、ログアップロード、シークレット取得などを行う際に使用、通常は最小権限のみ必要。
- Service Role は稀なシナリオで使用（Service Auto Scaling など）、ほとんどのシナリオで特別な設定不要。
- Container Instance Role は EC2 Mode でのみ使用、Fargate Mode では設定不要。

## Cluster Manager と Task 目標状態
- Cluster Manager は ECS の頭脳で、各種 Service、Task を対応するコンテナノードに「割り当て」て実行する責任があります。
- Task 目標状態：例えば Service に 100 個の Task が必要と設定した場合、Cluster Manager はどの特定の EC2 Instance や Fargate Node が実行するかは気にせず、現在生きている Task 数が目標に達しているかのみを関心。
  - Task が失敗すると、Cluster Manager は即座に新しい Task を補充して目標状態の一貫性を確保。
  - 実際の運用では、Task は異なる Instance（EC2 または Fargate ノード）に分散されますが、これはユーザーには透明。

## ELB と Auto Scaling Group の役割
| コンポーネント | 主な機能 |
| --------------------------- | -------------------------------------------------------- |
| ELB (Elastic Load Balancer) | コンテナサービスの外部トラフィック分散、単一の外部 Endpoint 提供 |
| Auto Scaling Group | EC2 Container Instance が Task 数に基づいて自動増減、トラフィックと Task 負荷に動的対応 |

#### クラウド環境での利点：
- 10万個の Task が必要な場合、Auto Scaling Group は設定した scaling policy に基づいて EC2 Instance を自動増減し、新しいマシンを ECS Cluster に自動登録。
- Fargate ではより便利で、Task 数を増やすだけで、AWS が十分なコンピューティングリソースを自動準備、Instance 管理と Scaling を考慮不要。

## 公式観点と一般的な誤解の補足
- Task Role の割り当て：各 ECS Task は異なる Task Role を持つことができ、Fargate/EC2 の両方でサポート、AWS が強調する最も細かい権限モデル（Least Privilege）。
- Fargate は Instance 管理不要：Fargate モードでは Container Instance Role の設定不要、権限は Task Role/Task Execution Role で決定、基盤 Instance は AWS が自己管理、見えず設定不可。
- 弾性拡張：Fargate は AWS のリソーススケジューリングで処理、EC2 は ASG scaling と Instance ライフサイクル管理を考慮必要。

## Fargate Mode での Auto Scaling について

### Fargate Mode の底層に実際に ASG はあるのか？
- ユーザー視点から：Fargate Mode では、Auto Scaling Group (ASG) や EC2 Instance を全く見ることも直接管理することもできません。
- AWS バックエンド実装：AWS は確かに内部でリソースプール（底層には依然として VM/ASG でスケジューリング）を持っていますが、この部分は完全に AWS が管理・最適化し、開発者には見えず設定もできません。
- 重要点：AWS Console、CLI、API では ASG 関連の設定や監視タブは一切見えません。ASG は AWS の「ブラックボックス」運用プロセスにのみ存在し、Task を実行するのに十分なリソースを保証。

### Fargate ユーザーがまだ設定できる scaling パラメータは？
- Service scaling：ECS Service の目標 Task 数量を設定し、メトリクス（CPU/Memory 使用率、Queue 長さなど）に基づく自動スケーリング（Auto Scaling）が可能。
  - これは「Task 数量」の調整であり、VM 数量の調整ではありません。
  - ECS は scaling policy に基づいて自動的に Task を開始/停止し、Fargate は自動的に基盤コンピューティングリソースを準備/解放。
- Scaling 感度（例：
  - Application Auto Scaling を使用して scaling policy を設定可能、例えば
    - 平均 CPU が 70% を 5 分間超えると、N 個の Task を追加
    - SQS キュー長が 100 を超えると、Task を追加
  - これらはすべて ECS Service レベルで設定され、基盤 VM/ASG を直接設定するものではありません。

### Fargate で「まだ」設定できる詳細
| 設定可能項目 | 設定可能 | 例 |
| ------------------------- | ----- | -------------------------------------------- |
| Task あたりの vCPU/Memory | ✅ | 0.25 vCPU / 0.5GB ~ 16 vCPU / 120GB |
| Task 数量 | ✅ | 例：Service 最小 1 最大 500 |
| Service scaling policy | ✅ | Application Auto Scaling、Alarm-based policy |
| IAM Task Role | ✅ | 各 Task に異なる IAM Role を指定可能 |
| VPC/Subnet/Security Group | ✅ | Fargate タスクをどの VPC/Subnet で実行するか選択可能 |
| EC2 Instance タイプ | ❌ |  |
| ASG scaling policy | ❌ |  |
| Instance IAM Role | ❌ |  |
| EC2 instance の Life Cycle | ❌ |  |

### 結論
- Fargate には「見える」ASG はなく、すべての弾性スケジューリング、基盤マシンは AWS ブラックボックスで自動処理。
- 調整できる弾性ルールは、Task または Service レベルの Auto Scaling のみ。
- scaling 感度、Trigger、最大/最小 task 数は決められますが、VM や ASG を直接管理することはできません。

## 重要なまとめ
- EC2 Mode は特殊なリソース、ネットワーク要件やコスト削減時に適用
  - Instance、権限、ネットワーク設定の自己管理が必要
- Fargate Mode はインフラ管理が不要で、完全にオンデマンド自動弾性コンピューティングに適用
  - Task と Service の設定のみ必要、その他は AWS が対応