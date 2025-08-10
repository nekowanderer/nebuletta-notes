# EKS Ingress 起動失敗の完全なトラブルシューティング事例

[English](../en/08_eks_ingress_startup_failed_case.md) | [繁體中文](../zh-tw/08_eks_ingress_startup_failed_case.md) | [日本語](../ja/08_eks_ingress_startup_failed_case.md) | [インデックスに戻る](../README.md)

---

## 背景
- 実験日：2025/08/10
- 難易度：🤬🤬🤬🤬
- 説明：EKS クラスターのデプロイ後、ALB Ingress Controller が Application Load Balancer を作成できず、Ingress リソースの ADDRESS フィールドが空白状態のまま。

---

## 初期症状

### アプリケーションの状態
```bash
$ kubectl get pods -n nebuletta-app-ns
NAME                                   READY   STATUS    RESTARTS   AGE
beta-app-deployment-7d7575984d-48jwp   1/1     Running   0          12m
beta-app-deployment-7d7575984d-g7bth   1/1     Running   0          12m
beta-app-deployment-7d7575984d-s695t   1/1     Running   0          12m
prod-app-deployment-68cc5659d9-2kb4t   1/1     Running   0          12m
prod-app-deployment-68cc5659d9-4w2gm   1/1     Running   0          12m
prod-app-deployment-68cc5659d9-crft7   1/1     Running   0          12m

$ kubectl get svc -n nebuletta-app-ns
NAME                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
beta-app-service-clusterip   ClusterIP   172.20.94.217   <none>        8080/TCP   11m
prod-app-service-clusterip   ClusterIP   172.20.23.79    <none>        8080/TCP   11m

$ kubectl get ingress -n nebuletta-app-ns
NAME           CLASS   HOSTS                   ADDRESS   PORTS   AGE
ingress-path   alb     eks-lab.nebuletta.com             80      11m
```

**重要な問題**：Ingress の ADDRESS フィールドが空白で、ALB が正常に作成されていないことを示している。

---

## トラブルシューティングプロセス

### 第1段階：初期エラー分析

まず、local の kubectl を更新して、正しいクラスターに接続することを確認：
```bash
# 自分の AWS 設定に従って region とクラスター名を調整
$ aws eks update-kubeconfig --region ap-northeast-1 --name dev-eks-cluster
```

Load Balancer Controller のログを確認：

```bash
$ kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller --tail=50
```

**エラー発見**：
```
{"level":"error","ts":"2025-08-10T06:31:17Z","msg":"Reconciler error","controller":"ingress","object":{"name":"nebuletta-eks-lab"},"namespace":"","name":"nebuletta-eks-lab","reconcileID":"adf58942-8802-4c8c-9063-d6c3dea9d5c9","error":"WebIdentityErr: failed to retrieve credentials\ncaused by: RequestError: send request failed\ncaused by: Post \"https://sts.ap-northeast-1.amazonaws.com/\": dial tcp: lookup sts.ap-northeast-1.amazonaws.com on 172.20.0.10:53: read udp 10.0.12.84:49309->172.20.0.10:53: read: connection refused"}
```

**分析**：Load Balancer Controller が AWS STS サービスに接続して認証情報を取得できない。

### 第2段階：VPC Endpoints の確認

STS VPC Endpoint が存在するかチェック：

```bash
$ aws ec2 describe-vpc-endpoints --region ap-northeast-1 --filters "Name=tag:Name,Values=dev-networking-sts-endpoint" | jq

{
  "VpcEndpoints": [
    {
      "VpcEndpointId": "vpce-0cb49d2450a183b3f",
      "VpcEndpointType": "Interface",
      "VpcId": "vpc-0ef535df4de8e80bb",
      "ServiceName": "com.amazonaws.ap-northeast-1.sts",
      "State": "available",
      "PrivateDnsEnabled": true,
      // ... その他の詳細情報
    }
  ]
}
```

**結果**：STS VPC Endpoint は存在し、`available` 状態だが、依然として接続できない。

### 第3段階：DNS 問題の詳細分析

DNS 解決をテスト：

```bash
$ kubectl run dns-test --image=busybox:1.28 --rm -it --restart=Never -- nslookup sts.ap-northeast-1.amazonaws.com
# DNS クエリが停止し、応答が得られない
```

CoreDNS の状態を確認：

```bash
$ kubectl get pods -n kube-system -l k8s-app=kube-dns
NAME                       READY   STATUS    RESTARTS   AGE
coredns-5c658475b5-4hkn8   0/1     Pending   0          53m
coredns-5c658475b5-pcjxv   0/1     Pending   0          53m
```

**根本原因発見**：CoreDNS ポッドが `Pending` 状態で、クラスター全体が DNS 解決を実行できない！

### 第4段階：CoreDNS スケジューリング問題分析

CoreDNS ポッドの詳細状態を確認：

```bash
$ kubectl describe pods -n kube-system -l k8s-app=kube-dns
```

**重要なエラーメッセージ**：
```
Events:
  Type     Reason            Age                    From               Message
  ----     ------            ----                   ----               -------
  Warning  FailedScheduling  14m (x4 over 30m)      default-scheduler  0/8 nodes are available: 8 node(s) had untolerated taint {eks.amazonaws.com/compute-type: fargate}. preemption: 0/8 nodes are available: 8 Preemption is not helpful for scheduling.
```

**問題の原因**：
- すべてのノードが Fargate ノードで、`eks.amazonaws.com/compute-type: fargate` taint を持っている
- CoreDNS ポッドに対応する toleration がなく、Fargate ノードでスケジュールできない

Fargate Profile 設定を確認：

```bash
$ aws eks describe-fargate-profile --region ap-northeast-1 --cluster-name dev-eks-cluster --fargate-profile-name dev-eks-cluster-fp-default | jq
```

**確認**：Fargate profile には確実に `kube-system` namespace が含まれているが、CoreDNS は依然としてスケジュールできない。

---

## 根本原因

### 主な問題チェーン
1. **CoreDNS に Fargate toleration がない** → CoreDNS ポッドがスケジュールできない
2. **DNS 解決の失敗** → すべての DNS クエリが失敗する  
3. **Load Balancer Controller が STS に接続できない** → AWS 認証情報を取得できない
4. **ALB を作成できない** → Ingress ADDRESS フィールドが空白
5. **アプリケーションが Ingress 経由でアクセスできない** → port-forward のみ使用可能

### 技術的原理
- **EKS Fargate** 環境では、すべてのポッドが Fargate ノードで実行される
- **Fargate ノード**には特別な taint がある：`eks.amazonaws.com/compute-type: fargate`
- **システムコンポーネント**（CoreDNS など）にはデフォルトで対応する toleration がない
- **ステートレス DNS** 問題がサービスチェーン全体の障害を引き起こす

---

## 解決策

### 解決ステップ1：CoreDNS Toleration の修正

CoreDNS deployment に Fargate toleration を追加：

```bash
$ kubectl patch deployment coredns -n kube-system --type='merge' -p='
{
  "spec": {
    "template": {
      "spec": {
        "tolerations": [
          {
            "key": "CriticalAddonsOnly",
            "operator": "Exists"
          },
          {
            "key": "node-role.kubernetes.io/control-plane",
            "effect": "NoSchedule"
          },
          {
            "key": "node.kubernetes.io/not-ready",
            "effect": "NoExecute",
            "tolerationSeconds": 300
          },
          {
            "key": "node.kubernetes.io/unreachable",
            "effect": "NoExecute", 
            "tolerationSeconds": 300
          },
          {
            "key": "eks.amazonaws.com/compute-type",
            "operator": "Equal",
            "value": "fargate",
            "effect": "NoSchedule"
          }
        ]
      }
    }
  }
}'

deployment.apps/coredns patched
```

**修正の検証**：
```bash
$ kubectl get pods -n kube-system -l k8s-app=kube-dns
NAME                      READY   STATUS    RESTARTS   AGE
coredns-56ffbd694-6xv6f   1/1     Running   0          71s
coredns-56ffbd694-ppz46   1/1     Running   0          71s
```

### 解決ステップ2：新しい問題の発見 - Subnet タグの不足

CoreDNS の修正後、Load Balancer Controller に新しいエラーが発生：

```bash
$ kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller --tail=20
```

**新しいエラーメッセージ**：
```
{"level":"error","ts":"2025-08-10T07:04:02Z","msg":"Reconciler error","controller":"ingress","object":{"name":"nebuletta-eks-lab"},"namespace":"","name":"nebuletta-eks-lab","reconcileID":"e4c5afd6-fb56-495e-9a7c-30c7450e2ecc","error":"couldn't auto-discover subnets: unable to resolve at least one subnet (0 match VPC and tags: [kubernetes.io/role/elb])"}
```

**問題分析**：Load Balancer Controller が ALB をデプロイする適切な subnet を見つけられない。

Subnet タグを確認：

```bash
# パブリック subnet を確認
$ aws ec2 describe-subnets --region ap-northeast-1 --filters "Name=vpc-id,Values=vpc-0ef535df4de8e80bb" --query 'Subnets[?MapPublicIpOnLaunch==`true`].[SubnetId,Tags]'

# プライベート subnet を確認  
$ aws ec2 describe-subnets --region ap-northeast-1 --filters "Name=vpc-id,Values=vpc-0ef535df4de8e80bb" --query 'Subnets[?MapPublicIpOnLaunch==`false`].[SubnetId,Tags]'
```

**問題発見**：Subnet に ALB に必要な Kubernetes タグがない：
- パブリック subnet に不足：`kubernetes.io/role/elb = 1`
- プライベート subnet に不足：`kubernetes.io/role/internal-elb = 1`

### 解決ステップ3：Subnet タグの追加

不足しているタグを手動で追加：

```bash
# パブリックサブネットにタグを追加
$ aws ec2 create-tags --region ap-northeast-1 --resources subnet-03137f56962d0a148 subnet-0a11863cea526b5a0 subnet-00b4e519a524eb1f3 --tags Key=kubernetes.io/role/elb,Value=1

# プライベートサブネットにタグを追加
$ aws ec2 create-tags --region ap-northeast-1 --resources subnet-00d1d5009e0fd2145 subnet-0dbcd8d5c45e993f5 subnet-087c8cf6e4ba0d2ae --tags Key=kubernetes.io/role/internal-elb,Value=1
```

### 解決結果の検証

タグ追加後、Load Balancer Controller が正常に ALB を作成：

```bash
$ kubectl get ingress -n nebuletta-app-ns
NAME           CLASS   HOSTS                   ADDRESS                                                                      PORTS   AGE
ingress-path   alb     eks-lab.nebuletta.com   k8s-nebulettaekslab-4b28eb94af-1752493976.ap-northeast-1.elb.amazonaws.com   80      42m
```

**成功ログ**：
```
{"level":"info","ts":"2025-08-10T07:09:35Z","logger":"controllers.ingress","msg":"created loadBalancer","stackID":"nebuletta-eks-lab","resourceID":"LoadBalancer","arn":"arn:aws:elasticloadbalancing:ap-northeast-1:362395300803:loadbalancer/app/k8s-nebulettaekslab-4b28eb94af/3e2e73703b34a339"}
{"level":"info","ts":"2025-08-10T07:09:35Z","logger":"controllers.ingress","msg":"created listener","stackID":"nebuletta-eks-lab","resourceID":"80","arn":"arn:aws:elasticloadbalancing:ap-northeast-1:362395300803:listener/app/k8s-nebulettaekslab-4b28eb94af/3e2e73703b34a339/d26a631eb694bf64"}
{"level":"info","ts":"2025-08-10T07:09:36Z","logger":"controllers.ingress","msg":"successfully deployed model","ingressGroup":"nebuletta-eks-lab"}
```

**アプリケーションテスト**：
```bash
$ curl k8s-nebulettaekslab-4b28eb94af-1752493976.ap-northeast-1.elb.amazonaws.com:80/beta -H 'Host: eks-lab.nebuletta.com'
[beta] served by: beta-app-deployment-7d7575984d-48jwp

$ curl k8s-nebulettaekslab-4b28eb94af-1752493976.ap-northeast-1.elb.amazonaws.com:80/prod -H 'Host: eks-lab.nebuletta.com'
[prod] served by: prod-app-deployment-68cc5659d9-4w2gm
```

---

## Terraform モジュール改善提案

### 1. EKS Addons を使用したシステムコンポーネント管理（推奨）

#### ベストプラクティス：公式 EKS Addons

実際の検証後、最良の解決策は AWS EKS の公式 addons を使用してシステムを管理することです。Kubernetes リソースを手動でパッチするのではなく、この方法はより安定で信頼性が高く、AWS によって完全に管理されます。

**`terraform/modules/eks-lab/cluster/addons.tf` を追加**：
```hcl
# EKS Addons 設定 - 基本システムコンポーネントを管理
# これらの addons は Fargate 互換性の問題を自動的に処理します

# VPC CNI addon - Pod ネットワーキングを処理
resource "aws_eks_addon" "vpc_cni" {
  cluster_name                = aws_eks_cluster.main.name
  addon_name                  = "vpc-cni"
  addon_version              = var.vpc_cni_addon_version
  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "PRESERVE"

  depends_on = [
    aws_eks_cluster.main,
    aws_eks_fargate_profile.default
  ]
}

# Kube-proxy addon - サービスネットワーキングを処理
resource "aws_eks_addon" "kube_proxy" {
  cluster_name                = aws_eks_cluster.main.name
  addon_name                  = "kube-proxy"
  addon_version              = var.kube_proxy_addon_version
  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "PRESERVE"

  depends_on = [
    aws_eks_cluster.main,
    aws_eks_fargate_profile.default
  ]
}

# CoreDNS addon - Fargate toleration を自動設定
resource "aws_eks_addon" "coredns" {
  cluster_name                = aws_eks_cluster.main.name
  addon_name                  = "coredns"
  addon_version              = var.coredns_addon_version
  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "PRESERVE"

  configuration_values = jsonencode({
    tolerations = [
      {
        key      = "CriticalAddonsOnly"
        operator = "Exists"
      },
      {
        key    = "node-role.kubernetes.io/control-plane"
        effect = "NoSchedule"
      },
      {
        key               = "node.kubernetes.io/not-ready"
        effect            = "NoExecute"
        tolerationSeconds = 300
      },
      {
        key               = "node.kubernetes.io/unreachable"
        effect            = "NoExecute"
        tolerationSeconds = 300
      },
      {
        key      = "eks.amazonaws.com/compute-type"
        operator = "Equal"
        value    = "fargate"
        effect   = "NoSchedule"
      }
    ]
  })

  depends_on = [
    aws_eks_cluster.main,
    aws_eks_fargate_profile.default
  ]
}

# Metrics Server addon - HPA などの機能をサポート
resource "aws_eks_addon" "metrics_server" {
  cluster_name                = aws_eks_cluster.main.name
  addon_name                  = "metrics-server"
  addon_version              = var.metrics_server_addon_version
  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "PRESERVE"

  configuration_values = jsonencode({
    tolerations = [
      {
        key      = "CriticalAddonsOnly"
        operator = "Exists"
      },
      {
        key    = "node-role.kubernetes.io/control-plane"
        effect = "NoSchedule"
      },
      {
        key               = "node.kubernetes.io/not-ready"
        effect            = "NoExecute"
        tolerationSeconds = 300
      },
      {
        key               = "node.kubernetes.io/unreachable"
        effect            = "NoExecute"
        tolerationSeconds = 300
      },
      {
        key      = "eks.amazonaws.com/compute-type"
        operator = "Equal"
        value    = "fargate"
        effect   = "NoSchedule"
      }
    ]
  })

  depends_on = [
    aws_eks_cluster.main,
    aws_eks_fargate_profile.default
  ]
}
```

**`terraform/modules/eks-lab/cluster/variables.tf` を修正**、バージョン変数を追加：
```hcl
# EKS Addons バージョン変数、バージョンは AWS EKS addons コンソールで確認可能
variable "vpc_cni_addon_version" {
  description = "VPC CNI addon version"
  type        = string
  default     = "v1.19.0-eksbuild.1"
}

variable "kube_proxy_addon_version" {
  description = "Kube-proxy addon version"
  type        = string
  default     = "v1.33.0-eksbuild.2"
}

variable "coredns_addon_version" {
  description = "CoreDNS addon version"
  type        = string
  default     = "v1.11.4-eksbuild.2"
}

variable "metrics_server_addon_version" {
  description = "Metrics Server addon version"
  type        = string
  default     = "v0.8.0-eksbuild.1"
}
```

#### なぜ EKS Addons を使用するのか？

1. **公式サポート**：AWS によって直接維持・管理されている
2. **自動更新**：互換性のあるバージョンに自動更新を設定可能
3. **Fargate 互換性**：toleration とスケジューリング要件を自動的に処理
4. **一貫性**：手動パッチで生じる可能性のある設定ドリフトを回避
5. **信頼性**：クロスプロバイダー操作の複雑さを軽減

#### 古い方法との比較

| 方法 | 手動 kubernetes_manifest パッチ | EKS Addons |
|------|--------------------------------|------------|
| 複雑さ | 高（クロスプロバイダー操作が必要）| 低（AWS ネイティブ管理）|
| 信頼性 | 中程度（実行順序に依存）| 高（AWS 保証）|
| 保守性 | 困難（手動更新が必要）| 簡単（自動更新可能）|
| エラー処理 | 複雑 | 組み込み競合解決 |

### 2. Subnet タグ修正

`networking` モジュールの `vpc.tf` を修正：

```hcl
# パブリックサブネット
resource "aws_subnet" "public" {
  count                   = length(local.public_subnet_cidrs)
  vpc_id                  = aws_vpc.this.id
  cidr_block              = local.public_subnet_cidrs[count.index]
  availability_zone       = local.available_azs[count.index]
  map_public_ip_on_launch = true
  
  tags = merge(local.common_tags, {
    Name                     = "${local.prefix}-public-${count.index}"
    "kubernetes.io/role/elb" = "1"  # このタグを追加
  })
}

# プライベートサブネット
resource "aws_subnet" "private" {
  count             = length(local.private_subnet_cidrs)
  vpc_id            = aws_vpc.this.id
  cidr_block        = local.private_subnet_cidrs[count.index]
  availability_zone = local.available_azs[count.index]
  
  tags = merge(local.common_tags, {
    Name                              = "${local.prefix}-private-${count.index}"
    "kubernetes.io/role/internal-elb" = "1"  # このタグを追加
  })
}
```

#### なぜタグ値は "1" でなければならないのか？

これは多くの人が疑問に思う重要な詳細です - なぜ `"true"` や他の値ではないのか：

**1. AWS 公式ドキュメントの要求**
- AWS Load Balancer Controller 公式ドキュメントでは、タグ値は `"1"` でなければならないと明確に述べています
- これは AWS と Kubernetes エコシステムの標準的な約束事です

**2. タグ検索メカニズム**
```bash
# AWS Load Balancer Controller は内部で以下のような検索を実行：
aws ec2 describe-subnets --filters "Name=tag:kubernetes.io/role/elb,Values=1"
aws ec2 describe-subnets --filters "Name=tag:kubernetes.io/role/internal-elb,Values=1"
```

**3. 値の比較**
| タグ値 | サポート状況 | 説明 |
|-------|-------------|------|
| `"1"` | ✅ 公式標準 | AWS 公式ドキュメントで要求される値 |
| `"true"` | ❌ サポートなし | 意味的には正しいが、Controller は認識しない |
| `"yes"` | ❌ サポートなし | 同様に Controller は認識しない |
| `""` (空値) | ⚠️ 部分サポート | 技術的には有効だが推奨されない |

**4. 歴史的背景**
- この慣例は初期の Kubernetes ラベルシステムに由来します
- ブール型マーキングにおいて、`"1"` は "enabled" または "true" を表します
- 後方互換性を維持するため、今日まで継続して使用されています

**5. 実際の検証**
間違ったタグ値を使用すると、Load Balancer Controller はエラーを報告します：
```
couldn't auto-discover subnets: unable to resolve at least one subnet (0 match VPC and tags: [kubernetes.io/role/elb])
```

**覚えておいてください**：必ず `"1"` を使用してください。これは任意の選択ではなく、AWS の厳格な要求事項です！

---

## 重要な学習ポイント

### EKS Fargate の特殊性

1. **Taint メカニズム**：Fargate ノードは特別な taint を持ち、対応する toleration が必要
2. **システムコンポーネント**：CoreDNS などのシステムコンポーネントは Fargate で実行するために追加設定が必要な場合がある
3. **DNS 依存性**：クラスター全体のネットワーク機能は CoreDNS の正常な動作に依存している

### ALB Ingress Controller の要件

1. **Subnet タグ**：用途に応じて subnet を正しくタグ付けする必要がある
   - `kubernetes.io/role/elb`：パブリック ALB 用
   - `kubernetes.io/role/internal-elb`：内部 ALB 用
2. **ネットワーク接続**：AWS API にアクセスできる必要がある（VPC endpoints または NAT Gateway 経由）
3. **IAM 権限**：IRSA で正しい IAM 権限を設定する必要がある

### 問題診断技術

1. **外側から内側へ**：ユーザーが見える症状から追跡を開始
2. **依存関係チェーンの確認**：各コンポーネントの依存サービスが正常かを確認
3. **階層診断**：底層（DNS）から上層（アプリケーション）まで段階的に確認
4. **ログ分析**：kubectl logs を効果的に使用して具体的なエラーメッセージを見つける

### 補足
- 実際には、AWS EKS 実践ガイドの [EKS クラスター作成](../../containerization/orchestration/Kubernetes/ja/35_eks_practice.md#eks-クラスター作成) セクションで、以下のログが確認できます：
  ```bash
  default addons vpc-cni, kube-proxy, coredns, metrics-server were not specified, will install them as EKS addons
  ...
  creating addon: coredns
  successfully created addon: coredns
  ...
  "coredns" is now schedulable onto Fargate
  "coredns" is now scheduled onto Fargate
  "coredns" pods are now scheduled onto Fargate
  ```
- これは以下のことを示しています：
  1. EKS Addon 管理：eksctl は単純な Kubernetes Deployment ではなく、EKS の addon システムを使用して CoreDNS をデプロイします
  2. Fargate 認識：`eksctl create cluster --fargate` を使用すると、eksctl は：
    - デフォルトの Fargate Profile を自動作成
    - CoreDNS が Fargate でスケジュール可能になるよう自動設定
    - 必要な toleration 設定を処理
  3. Terraform との違い：Terraform で作成するのは：
    - ネイティブな EKS クラスター
    - 手動で作成された Fargate Profile
    - CoreDNS は依然として EKS のデフォルトデプロイメントだが、自動 Fargate スケジューリング設定なし
- これが、eksctl を通じて手動操作した際にこの問題に遭遇しなかった理由です。AWS が事前に処理していたからです

---

## 予防策

1. **モジュール設計**：インフラストラクチャモジュールで最初から EKS の特別な要件を考慮
2. **自動テスト**：デプロイ後に DNS 解決と基本機能を自動検証
3. **監視アラート**：CoreDNS と Load Balancer Controller のヘルスチェックを設定
4. **ドキュメント化**：既知の問題と解決策を記録し、重複した問題を回避

---

## 重要な注意事項：Terraform Output と ALB 準備タイミング

### ALB 作成の非同期特性

applications stack のデプロイ後、以下の状況に遭遇する可能性があります：

```bash
# kubectl は ingress が ADDRESS を持っていることを表示
$ kubectl get ingress -n nebuletta-app-ns
NAME           CLASS   HOSTS                   ADDRESS
ingress-path   alb     eks-lab.nebuletta.com   k8s-nebulettaekslab-4b28eb94af-1249260893.ap-northeast-1.elb.amazonaws.com

# しかし terraform output は空値を表示
$ terramate run --tags dev-eks-applications -- terraform output
ingress_address = ""
ingress_hostname = ""
```

### 原因分析

これは以下の理由によるものです：
1. **ALB 作成は非同期プロセス**：Terraform apply が完了した時点で、ALB はまだ作成中の可能性がある
2. **Terraform 状態スナップショット**：output は最後にリソース状態を読み取った時の値を表示
3. **Kubernetes リアルタイム状態**：kubectl はクラスターのリアルタイム状態を表示

### 解決方法

**方法1：Terraform 状態の更新（推奨）**
```bash
# 最新の ingress 情報を取得するために状態を更新
terramate run --tags dev-eks-applications -- terraform refresh

# 再度 output を確認
terramate run --tags dev-eks-applications -- terraform output
```

**方法2：apply の再実行**
```bash
# apply を再実行すると自動的に状態が更新される
terramate run --tags dev-eks-applications -- terraform apply -auto-approve

# output を確認
terramate run --tags dev-eks-applications -- terraform output
```

**方法3：継続監視**
```bash
# 30秒ごとに確認
watch -n 30 "terramate run --tags dev-eks-applications -- terraform output ingress_address"
```

### 時間予想

- **ALB 作成時間**：通常 2-5 分必要
- **DNS 伝播時間**：追加で 1-2 分
- **合計待機時間**：約 3-7 分

### ALB 準備状態の検証

```bash
# 方法1：Kubernetes ingress 状態を確認
kubectl get ingress -n nebuletta-app-ns

# 方法2：AWS Load Balancer Controller ログを確認
kubectl logs -n kube-system deployment/aws-load-balancer-controller

# 方法3：DNS 解決をテスト
dig +short k8s-nebulettaekslab-4b28eb94af-1249260893.ap-northeast-1.elb.amazonaws.com
```

### 注意

**急いでトラブルシューティングしない**：kubectl で ingress に ADDRESS があるが terraform output が空の場合、**まず ALB が完全に準備されるのを待ち**、その後 `terraform refresh` を実行してください。これは正常な非同期動作であり、設定エラーではありません。

---

## 結論

今回のトラブルシューティング経験は、EKS Fargate 環境の複雑さを示しています。単純に見える「Ingress が ALB を作成できない」問題が、実際には以下を含んでいました：

1. **CoreDNS スケジューリング問題**（Fargate toleration）
2. **DNS 解決の失敗**（クラスター全体のネットワーク機能中断）
3. **AWS API アクセス問題**（STS 接続失敗）
4. **Subnet タグの不足**（ALB が適切な subnet を見つけられない）

**重要なポイント**：EKS Fargate 環境では、システムコンポーネントの正しい設定がクラスター全体の機能の基礎となります。CoreDNS は DNS サービスの中核として、その障害は連鎖反応を引き起こし、DNS クエリが必要なすべてのサービスに影響を与えます。問題の表象は往々にして根本原因ではなく、体系的な分析によって真の根本問題を見つける必要があります。
