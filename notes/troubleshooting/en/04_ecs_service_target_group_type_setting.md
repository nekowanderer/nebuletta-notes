# Networking Options for ECS Tasks under Fargate Launch Type

[English](../en/04_ecs_service_target_group_type_setting.md) | [繁體中文](../zh-tw/04_ecs_service_target_group_type_setting.md) | [日本語](../ja/04_ecs_service_target_group_type_setting.md) | [Back to Index](../README.md)

---

## Background
- Experiment Date: 2025/07/16
- Difficulty: 🤬
- Description: When creating an ECS service using Terraform, the binding between ALB listener and target group failed.

---

## Problem Encountered

- When creating an ECS service through Terraform, the following error occurred:

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
- It appears that the target type was not configured correctly.

---

## Root Cause

- The target group was configured as follows:
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
- No target_type was defined. According to Terraform's [official documentation](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_target_group), the default is `instance`. However, the ECS task definition was configured as:

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

- We need to understand the concept of ECS network modes:

#### ECS Network Mode Differences

- EC2 Launch Type (traditional mode):
  - Uses bridge or host network mode
  - Containers run on EC2 instances
  - Target Group target_type = `instance`
  - ALB routes traffic to EC2 instances, then instances forward to containers

- Fargate Launch Type:
  - Mandatorily uses awsvpc network mode
  - Each task has its own ENI (Elastic Network Interface)
  - Each task has an independent private IP address
  - Target Group must use target_type = `ip`
  - ALB routes traffic directly to task IP addresses

- Why Fargate requires target_type = `ip`
  - Network isolation: Fargate tasks have independent network namespaces
  - Dynamic IP: Tasks receive dynamically assigned IPs when launched
  - Direct routing: ALB needs to route directly to task IPs, not through EC2 instances

- Documentation Reference
  - Although Terraform documentation doesn't explicitly explain this relationship, AWS [official documentation](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-task-networking.html) states:
> ⚠️ **Services with tasks that use the Fargate launch type only support Application Load Balancer and Network Load Balancer. Classic Load Balancer isn't supported. When you create any target groups, you must choose ip as the target type, not instance.**

#### Complete Causal Relationship

1. Task Definition setting: `network_mode = "awsvpc"`
2. Fargate requirement: `requires_compatibilities = ["FARGATE"]` forces `awsvpc`
3. Result: Target Group must use `target_type = "ip"`

- Mapping of three scenarios
  | Launch Type | Network Mode | Target Type | Reason              |
  |-------------|--------------|-------------|---------------------|
  | EC2         | bridge/host  | instance    | Traffic routed to EC2 instances |
  | EC2         | awsvpc       | ip          | Tasks have independent IPs      |
  | Fargate     | awsvpc (mandatory) | ip          | Tasks have independent IPs     |

---

## Solution
- Simply add `target_type = "ip"`:

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

## Summary
- Housing analogy
  | Item          | EC2 + bridge | Fargate + awsvpc |
  |-------------|--------------|------------------|
  | Housing Type        | Apartment Building       | Detached House         |
  | Address Type        | Unified Building Address       | Individual House Number           |
  | Mail Delivery        | Delivered to building→staff distributes   | Directly to address         |
  | Target Type | instance     | ip               |

- Courier delivery analogy
  | Role   | EC2 Mode          | Fargate Mode |
  |------|-----------------|------------|
  | ALB  | Courier          | Courier     |
  | Delivery Method | Deliver to building management office         | Deliver directly to resident's door  |
  | Intermediary  | Staff responsible for distribution      | No intermediary    |
  | Address Type | Building address (instance) | Resident address (ip)  |

- Conclusion
  | Network Mode   | Like    | Target Type | Reason         |
  |--------|-------|-------------|------------|
  | bridge | Living in apartment building   | instance    | Unified address, staff distributes |
  | awsvpc | Living in independent apartment | ip          | Independent address, direct delivery  |