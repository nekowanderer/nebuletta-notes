# Fargate Launch Type における ECS Task のネットワークオプション

[English](../en/04_ecs_service_target_group_type_setting.md) | [繁體中文](../zh-tw/04_ecs_service_target_group_type_setting.md) | [日本語](../ja/04_ecs_service_target_group_type_setting.md) | [インデックスに戻る](../README.md)

---

## 背景
- 実験日：2025/07/16
- 難易度：🤬
- 説明：TerraformでECSサービスを作成する際、ALBリスナーとターゲットグループの結合が失敗した。

---

## 遭遇した問題

- Terraformを通じてECSサービスを作成する際、以下のエラーが発生した：

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
- target typeが正しく設定されていないようだ。

---

## エラーの原因

- この時のターゲットグループは以下のように書かれていた：
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
- target_typeが定義されておらず、Terraformの[公式ドキュメント](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_target_group)によると、デフォルトは`instance`となっているが、この時のECSタスク定義は以下のように定義されていた：

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

- まず、ECSネットワークモードの概念を理解する必要がある：

#### ECSネットワークモードの違い

- EC2 Launch Type（従来モード）：
  - bridgeまたはhostネットワークモードを使用
  - コンテナはEC2インスタンス上で実行
  - ターゲットグループのtarget_type = `instance`
  - ALBはEC2インスタンスにトラフィックをルーティングし、その後インスタンスがコンテナに転送

- Fargate Launch Type：
  - awsvpcネットワークモードを強制的に使用
  - 各タスクは独自のENI（Elastic Network Interface）を持つ
  - 各タスクは独立したプライベートIPアドレスを持つ
  - ターゲットグループはtarget_type = `ip`を使用する必要がある
  - ALBはタスクのIPアドレスに直接トラフィックをルーティング

- FargateがtargetType = `ip`を必要とする理由
  - ネットワーク分離：Fargateタスクは独立したネットワーク名前空間を持つ
  - 動的IP：タスクは起動時に動的に割り当てられたIPを取得
  - 直接ルーティング：ALBはEC2インスタンスを通さずにタスクのIPに直接ルーティングする必要がある

- ドキュメント参照
  - Terraformドキュメントではこの関係について明確に説明されていないが、AWS[公式ドキュメント](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-task-networking.html)には以下のように記載されている：
> ⚠️ **Services with tasks that use the Fargate launch type only support Application Load Balancer and Network Load Balancer. Classic Load Balancer isn't supported. When you create any target groups, you must choose ip as the target type, not instance.**

#### 完全な因果関係

1. タスク定義設定：`network_mode = "awsvpc"`
2. Fargate要求：`requires_compatibilities = ["FARGATE"]`が`awsvpc`を強制
3. 結果：ターゲットグループは`target_type = "ip"`を使用する必要がある

- 3つのシナリオの対応関係
  | Launch Type | Network Mode | Target Type | 理由              |
  |-------------|--------------|-------------|------------------|
  | EC2         | bridge/host  | instance    | トラフィックがEC2インスタンスにルーティング |
  | EC2         | awsvpc       | ip          | タスクが独立したIPを持つ      |
  | Fargate     | awsvpc（強制） | ip          | タスクが独立したIPを持つ     |

---

## 解決方法
- `target_type = "ip"`を追加するだけで解決：

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

## まとめ
- 住居の比喩
  | 項目          | EC2 + bridge | Fargate + awsvpc |
  |-------------|--------------|------------------|
  | 住居方式        | アパートビル       | 一戸建て住宅         |
  | 住所方式        | ビル統一住所       | 各戸独立番地           |
  | 郵便配達        | ビルに配達→管理人が配布   | 直接住所に配達         |
  | Target Type | instance     | ip               |

- 宅配便の比喩
  | 役割   | EC2モード          | Fargateモード |
  |------|-----------------|------------|
  | ALB  | 配達員          | 配達員     |
  | 配達方法 | ビルの管理室に配達         | 直接住人の玄関に配達  |
  | 仲介人  | 管理人が配布を担当      | 仲介人なし    |
  | 住所タイプ | ビル住所（instance） | 住人住所（ip）  |

- 結論
  | ネットワークモード   | 例えるなら    | Target Type | 理由         |
  |--------|-------|-------------|------------|
  | bridge | アパートビル居住   | instance    | 統一住所、管理人が配布 |
  | awsvpc | 独立アパート居住 | ip          | 独立番地、直接配達  |