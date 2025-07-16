# Fargate Lunch Type 之下的 ECS Task 的 Networking Options

[English](../en/04_ecs_service_target_group_type_setting.md) | [繁體中文](./04_ecs_service_target_group_type_setting.md) | [日本語](../ja/04_ecs_service_target_group_type_setting.md) | [返回索引](../README.md)

---

## 背景
- 實驗日期：2025/07/16
- 難度：🤬
- 描述：使用 Terraform 建立 ECS service 的時候，ALB listener 與 target group 的綁定失敗了。

---

## 遇到的問題

- 在透過 terraform 建立 ecs-service 時，出現了以下錯誤：

```bash
module.ecs_service.aws_ecs_service.beta_service: Creating...
╷
│ Error: creating ECS Service (my-service-beta): operation error ECS: CreateService, https response error StatusCode: 400, RequestID: 9d354d42-cfb1-4809-ab1e-dfffee5d0924, InvalidParameterException: The provided target group arn:aws:elasticloadbalancing:ap-northeast-1:362395300803:targetgroup/my-tg-beta/69604aca2eca0d4f has target type instance, which is incompatible with the awsvpc network mode specified in the task definition.
│
│   with module.ecs_service.aws_ecs_service.beta_service,
│   on ../../../modules/ecs-service/ecs-service.tf line 41, in resource "aws_ecs_service" "beta_service":
│   41: resource "aws_ecs_service" "beta_service" {
│
╵
Error: one or more commands failed
> execution failed: running /opt/homebrew/bin/terraform apply (in /terraform/stacks/dev/ecs-service): exit status 1
```
- 看來是 target type 沒有正確設定。

---

## 錯誤的原因

- 此時的 target group 是這樣寫的：
```hcl
resource "aws_lb_target_group" "beta_tg" {
  name        = "my-tg-beta"
  port        = 80
  protocol    = "HTTP"
  vpc_id      = var.vpc_id

  health_check {
    enabled             = true
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 30
    path                = "/health"
    matcher             = "200"
  }

  tags = merge(local.common_tags, {
    Name = "${local.prefix}-beta-tg"
  })
}
```
- 沒有定義 target_type，根據 Terraform 的[官方文件](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_target_group)說明，預設就是 `instance`，然而，此時的 ECS task definition 是這樣定義的：

```hcl
resource "aws_ecs_task_definition" "beta_task" {
  family                   = local.beta_task_definition_family
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = var.cpu
  memory                   = var.memory
  execution_role_arn       = aws_iam_role.ecs_task_execution_role.arn
...
}
```

- 再來就要先了解 ECS 網路模式的概念：

#### ECS 網路模式的差異

- EC2 Launch Type (傳統模式)：
  - 使用 bridge 或 host 網路模式
  - 容器運行在 EC2 實例上
  - Target Group 的 target_type = `instance`
  - ALB 將流量路由到 EC2 實例，然後由實例轉發到容器

- Fargate Launch Type：
  - 強制使用 awsvpc 網路模式
  - 每個任務都有自己的 ENI (Elastic Network Interface)
  - 每個任務都有獨立的私有 IP 地址
  - Target Group 必須使用 target_type = `ip`
  - ALB 直接將流量路由到任務的 IP 地址

- 為什麼 Fargate 需要 target_type = `ip`
  - 網路隔離：Fargate 任務有獨立的網路命名空間
  - 動態 IP：任務啟動時會獲得動態分配的 IP
  - 接路由：ALB 需要直接路由到任務的 IP，而不是通過 EC2 實例

- 文件參考
  - 雖然 Terraform 文件沒有明確說明這個關係，但在 AWS [官方文件](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-task-networking.html)中有說明：
> ⚠️ **Services with tasks that use the Fargate launch type only support Application Load Balancer and Network Load Balancer. Classic Load Balancer isn't supported. When you create any target groups, you must choose ip as the target type, not instance.**

#### 完整的因果關係

1. Task Definition 設定：`network_mode = "awsvpc"`
2. Fargate 要求：`requires_compatibilities = ["FARGATE"]` 強制使用 `awsvpc`
3. 結果：Target Group 必須使用 `target_type = "ip"`

- 三種情況的對應關係
  | Launch Type | Network Mode | Target Type | 原因              |
  |-------------|--------------|-------------|------------------|
  | EC2         | bridge/host  | instance    | 流量路由到 EC2 實例 |
  | EC2         | awsvpc       | ip          | 任務有獨立 IP      |
  | Fargate     | awsvpc (強制) | ip          | 任務有獨立 IP     |

---

## 解決方式
- 補上 `target_type = "ip"` 即可：

```hcl
resource "aws_lb_target_group" "beta_tg" {
  name        = "my-tg-beta"
  port        = 80
  protocol    = "HTTP"
  vpc_id      = var.vpc_id
  target_type = "ip"

  health_check {
    enabled             = true
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 30
    path                = "/health"
    matcher             = "200"
  }

  tags = merge(local.common_tags, {
    Name = "${local.prefix}-beta-tg"
  })
}
```

---

## 小結
- 用住宿的方式比喻
  | 項目          | EC2 + bridge | Fargate + awsvpc |
  |-------------|--------------|------------------|
  | 住宿方式        | 住大樓       | 住獨棟一戶建         |
  | 地址方式        | 大樓統一地址       | 每戶獨立門牌           |
  | 郵件配送        | 寄到大樓→管理員分發   | 直接寄到地址         |
  | Target Type | instance     | ip               |

- 用快遞配送比喻
  | 角色   | EC2 模式          | Fargate 模式 |
  |------|-----------------|------------|
  | ALB  | 快遞員          | 快遞員     |
  | 配送方式 | 送到大樓管理室         | 直接送到住戶家門口  |
  | 中間人  | 管理員負責分發      | 沒有中間人    |
  | 地址類型 | 大樓地址 (instance) | 住戶地址 (ip)  |

- 結論
  | 網路模式   | 就像    | Target Type | 原因         |
  |--------|-------|-------------|------------|
  | bridge | 住大樓   | instance    | 統一地址，管理員分發 |
  | awsvpc | 住獨立公寓 | ip          | 獨立門牌，直接配送  |
