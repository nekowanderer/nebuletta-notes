# AWS ECS: Differences Between EC2 Mode and Fargate Mode

[English](../en/18_aws_ecs.md) | [繁體中文](../zh-tw/18_aws_ecs.md) | [日本語](../ja/18_aws_ecs.md) | [Back to Index](../README.md)

## ECS Basic Architecture Overview
- ECS (Elastic Container Service) is a container management platform provided by AWS.
- Basic concept: You package your applications into Docker Images and hand them over to ECS. ECS helps you start, manage, and monitor these containers on backend compute resources (EC2 or Fargate).

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

## ECS EC2 Mode vs Fargate Mode Overview
| Feature | EC2 Mode | Fargate Mode |
| -------- | -------------------------------- | ------------------------ |
| Compute Resource Management | You create/manage EC2 Instances yourself | AWS automatically manages, no VM handling needed |
| Self-managed Flexibility | Need to configure Auto Scaling Group, ELB | Automatic elastic scaling, no underlying resource management |
| IAM Permission Details | Can control IAM for EC2/Container Instance | Only focus on Task, Service IAM |
| Compute Resource Scaling | Need to manage EC2 quantity, manual/auto scaling | Only define Task quantity, Fargate automatically handles |
| Cost Structure | Billed per EC2 Instance | Billed per Task's required vCPU/Memory |
| Use Cases | Special network requirements, fine infrastructure control | Want simplicity, no infrastructure management |


## Container Instance Structure Analysis (EC2 Mode)
| Component | Description |
| ------------------- | ---------------------------------- |
| Linux | Underlying OS, usually Amazon Linux 2/Ubuntu |
| Docker Engine | Container runtime environment, responsible for starting/stopping containers |
| ECS Container Agent | Communicates with ECS Cluster Manager, manages containers on local machine |
- ECS Container Agent is a Daemon program provided by AWS, pre-installed on AWS official ECS AMI.
- Its main responsibility: Enable EC2 to receive commands from ECS Cluster Manager (such as which Task to start, status reporting, etc.).

## IAM Role Types and Differences
| Role Type | Authorization Scope | Official Purpose Description |
| ----------------------- | ------------------------ | -------------------------------------------- |
| Service Role | ECS Service | Provides Service access to AWS resources (e.g., Auto Scaling, ELB) |
| Task Execution Role | ECS Task Definition | Allows ECS to pull ECR images, write CloudWatch Logs when executing containers |
| Task Role | Running Container/Task | Provides AWS resource access permissions needed by Task (application) (S3, DynamoDB, etc.) |
| Container Instance Role | Container Instance (EC2) | For ECS Agent/EC2 itself to communicate with ECS management |

#### Official Documentation Supplement:
- Task Role is AWS permissions directly assigned to application containers, only used by containers within that Task (e.g., accessing S3, DynamoDB).
- Task Execution Role is used by ECS Agent to pull images, upload logs, retrieve secrets, etc., usually only requires minimal permissions.
- Service Role is used in rare scenarios (e.g., Service Auto Scaling), most scenarios don't require special configuration.
- Container Instance Role is only used in EC2 Mode, not needed in Fargate Mode.

## Cluster Manager and Task Target State
- Cluster Manager is the brain of ECS, responsible for "assigning" various Services and Tasks to corresponding container nodes for execution.
- Task target state: For example, if you configure a Service to need 100 Tasks, Cluster Manager doesn't care which specific EC2 Instances or Fargate Nodes execute them, only cares whether the current number of live Tasks meets the target.
  - If a Task fails, Cluster Manager immediately starts a new Task to ensure target state consistency.
  - During actual operation, Tasks are distributed across different Instances (EC2 or Fargate nodes), but this is transparent to you.

## Role of ELB and Auto Scaling Group
| Component | Main Function |
| --------------------------- | -------------------------------------------------------- |
| ELB (Elastic Load Balancer) | Distributes external traffic for container services, provides single external Endpoint |
| Auto Scaling Group | Allows EC2 Container Instances to automatically scale based on Task quantity, dynamically adapting to traffic and Task load |

#### Advantages in Cloud Environment:
- If you need 100,000 Tasks, Auto Scaling Group can automatically increase/decrease EC2 Instances based on your scaling policy settings, and automatically register new machines to ECS Cluster.
- More convenient with Fargate, you just increase Task quantity, AWS automatically prepares sufficient compute resources without considering Instance management and Scaling.

## Official Perspective and Common Misconceptions
- Task Role assignment: Each ECS Task can have different Task Roles, supported by both Fargate/EC2, also AWS's emphasized finest permission model (Least Privilege).
- Fargate requires no Instance management: In Fargate mode, no need to configure Container Instance Role, permissions determined by Task Role/Task Execution Role, underlying Instances managed by AWS, invisible and non-customizable.
- Elastic scaling: Fargate handled by AWS resource scheduling, EC2 requires considering ASG scaling and Instance lifecycle management.

## About Auto Scaling in Fargate Mode

### Does Fargate Mode actually have ASG underneath?
- From user perspective: In Fargate Mode, you cannot see or directly manage Auto Scaling Group (ASG) or EC2 Instances at all.
- AWS backend implementation: AWS indeed has resource pools internally (still has VM/ASG for scheduling underneath), but this part is completely managed and optimized by AWS, invisible and non-configurable for developers.
- Key point: You cannot see any ASG-related settings or monitoring tabs in AWS Console, CLI, API. ASG only exists in AWS's "black box" operations process, ensuring sufficient resources to execute your Tasks.

### What scaling parameters can Fargate users still configure?
- Service scaling: You can set target Task quantity for ECS Service, and auto scaling based on metrics (such as CPU/Memory usage, Queue length, etc.).
  - This adjusts "Task quantity", not VM quantity.
  - ECS automatically starts/stops Tasks based on scaling policy, Fargate automatically prepares/releases underlying compute resources.
- Scaling sensitivity (for example:
  - You can use Application Auto Scaling to set scaling policy, such as
    - When average CPU exceeds 70% for 5 minutes, add N Tasks
    - When SQS queue length exceeds 100, add Tasks
  - These are all configured at ECS Service level, not directly configuring underlying VM/ASG.

### Details you can "still" configure in Fargate
| Configurable Item | Can Configure | Example |
| ------------------------- | ----- | -------------------------------------------- |
| vCPU/Memory per Task | ✅ | 0.25 vCPU / 0.5GB ~ 16 vCPU / 120GB |
| Task Quantity | ✅ | e.g., Service minimum 1 maximum 500 |
| Service scaling policy | ✅ | Application Auto Scaling, Alarm-based policy |
| IAM Task Role | ✅ | Each Task can specify different IAM Role |
| VPC/Subnet/Security Group | ✅ | Can choose which VPC/Subnet Fargate tasks run on |
| EC2 Instance Type | ❌ |  |
| ASG scaling policy | ❌ |  |
| Instance IAM Role | ❌ |  |
| EC2 instance Life Cycle | ❌ |  |

### Conclusion
- Fargate has no "visible" ASG, all elastic scheduling and underlying machines handled automatically by AWS black box.
- The only elastic rules you can adjust are at Task or Service level Auto Scaling.
- You can still decide scaling sensitivity, Triggers, and maximum/minimum task counts, but cannot directly manage VM or ASG.

## Key Summary
- EC2 Mode suitable for special resource, network requirements or cost savings
  - Need to self-manage Instance, permissions and network configuration
- Fargate Mode suitable for no infrastructure management needs, fully on-demand automatic elastic computing
  - Only need to configure Task and Service, AWS handles everything else