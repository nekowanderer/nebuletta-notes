# EKS Ingress 啟動失敗的完整除錯案例

[English](../en/08_eks_ingress_startup_failed_case.md) | [繁體中文](../zh-tw/08_eks_ingress_startup_failed_case.md) | [日本語](../ja/08_eks_ingress_startup_failed_case.md) | [回到索引](../README.md)

---

## 背景
- 實驗日期：2025/08/10
- 難度：🤬🤬🤬🤬
- 描述：EKS cluster 部署後，ALB Ingress Controller 無法建立 Application Load Balancer，Ingress 資源的 ADDRESS 欄位持續為空白狀態。

---

## 初始現象

### 應用程式狀態
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

**關鍵問題**：Ingress 的 ADDRESS 欄位為空白，表示 ALB 沒有成功建立。

---

## 除錯過程

### 第一階段：初步錯誤分析

先更新一下 local 的 kubectl，確保接下來的指令是連到正確的 cluster：
```bash
# 根據自己的 AWS 設定去調整 region 跟 cluster name
$ aws eks update-kubeconfig --region ap-northeast-1 --name dev-eks-cluster
```

檢查 Load Balancer Controller 日誌：

```bash
$ kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller --tail=50
```

**發現錯誤**：
```
{"level":"error","ts":"2025-08-10T06:31:17Z","msg":"Reconciler error","controller":"ingress","object":{"name":"nebuletta-eks-lab"},"namespace":"","name":"nebuletta-eks-lab","reconcileID":"adf58942-8802-4c8c-9063-d6c3dea9d5c9","error":"WebIdentityErr: failed to retrieve credentials\ncaused by: RequestError: send request failed\ncaused by: Post \"https://sts.ap-northeast-1.amazonaws.com/\": dial tcp: lookup sts.ap-northeast-1.amazonaws.com on 172.20.0.10:53: read udp 10.0.12.84:49309->172.20.0.10:53: read: connection refused"}
```

**分析**：Load Balancer Controller 無法連接 AWS STS 服務取得憑證。

### 第二階段：檢查 VPC Endpoints

檢查 STS VPC Endpoint 是否存在：

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
      // ... 其他詳細資訊
    }
  ]
}
```

**結果**：STS VPC Endpoint 存在且為 `available` 狀態，但仍然無法連接。

### 第三階段：深入 DNS 問題分析

測試 DNS 解析：

```bash
$ kubectl run dns-test --image=busybox:1.28 --rm -it --restart=Never -- nslookup sts.ap-northeast-1.amazonaws.com
# DNS 查詢卡住，無法取得回應
```

檢查 CoreDNS 狀態：

```bash
$ kubectl get pods -n kube-system -l k8s-app=kube-dns
NAME                       READY   STATUS    RESTARTS   AGE
coredns-5c658475b5-4hkn8   0/1     Pending   0          53m
coredns-5c658475b5-pcjxv   0/1     Pending   0          53m
```

**根本問題發現**：CoreDNS pods 處於 `Pending` 狀態，導致整個叢集無法進行 DNS 解析！

### 第四階段：CoreDNS 調度問題分析

檢查 CoreDNS pods 詳細狀態：

```bash
$ kubectl describe pods -n kube-system -l k8s-app=kube-dns
```

**關鍵錯誤訊息**：
```
Events:
  Type     Reason            Age                    From               Message
  ----     ------            ----                   ----               -------
  Warning  FailedScheduling  14m (x4 over 30m)      default-scheduler  0/8 nodes are available: 8 node(s) had untolerated taint {eks.amazonaws.com/compute-type: fargate}. preemption: 0/8 nodes are available: 8 Preemption is not helpful for scheduling.
```

**問題原因**：
- 所有節點都是 Fargate nodes，具有 `eks.amazonaws.com/compute-type: fargate` taint
- CoreDNS pods 缺少對應的 toleration，無法在 Fargate nodes 上調度

檢查 Fargate Profile 設定：

```bash
$ aws eks describe-fargate-profile --region ap-northeast-1 --cluster-name dev-eks-cluster --fargate-profile-name dev-eks-cluster-fp-default | jq
```

**確認**：Fargate profile 確實包含 `kube-system` namespace，但 CoreDNS 仍無法調度。

---

## 問題根因

### 主要問題鏈
1. **CoreDNS 缺少 Fargate toleration** → CoreDNS pods 無法調度
2. **DNS 解析失敗** → 所有 DNS 查詢都會失敗  
3. **Load Balancer Controller 無法連接 STS** → 無法取得 AWS 憑證
4. **ALB 無法建立** → Ingress ADDRESS 欄位空白
5. **應用程式無法透過 Ingress 存取** → 只能使用 port-forward

### 技術原理
- **EKS Fargate** 環境中，所有 pods 都在 Fargate nodes 上運行
- **Fargate nodes** 具有特殊的 taint：`eks.amazonaws.com/compute-type: fargate`
- **系統元件**（如 CoreDNS）預設沒有對應的 toleration
- **Stateless DNS** 問題導致整個服務鏈失敗

---

## 解決方案

### 解決步驟一：修復 CoreDNS Toleration

為 CoreDNS deployment 新增 Fargate toleration：

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

**驗證修復**：
```bash
$ kubectl get pods -n kube-system -l k8s-app=kube-dns
NAME                      READY   STATUS    RESTARTS   AGE
coredns-56ffbd694-6xv6f   1/1     Running   0          71s
coredns-56ffbd694-ppz46   1/1     Running   0          71s
```

### 解決步驟二：發現新問題 - Subnet 標籤缺失

CoreDNS 修復後，Load Balancer Controller 出現新的錯誤：

```bash
$ kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller --tail=20
```

**新錯誤訊息**：
```
{"level":"error","ts":"2025-08-10T07:04:02Z","msg":"Reconciler error","controller":"ingress","object":{"name":"nebuletta-eks-lab"},"namespace":"","name":"nebuletta-eks-lab","reconcileID":"e4c5afd6-fb56-495e-9a7c-30c7450e2ecc","error":"couldn't auto-discover subnets: unable to resolve at least one subnet (0 match VPC and tags: [kubernetes.io/role/elb])"}
```

**問題分析**：Load Balancer Controller 找不到適當的 subnet 來部署 ALB。

檢查 subnet 標籤：

```bash
# 檢查公共 subnet
$ aws ec2 describe-subnets --region ap-northeast-1 --filters "Name=vpc-id,Values=vpc-0ef535df4de8e80bb" --query 'Subnets[?MapPublicIpOnLaunch==`true`].[SubnetId,Tags]'

# 檢查私有 subnet  
$ aws ec2 describe-subnets --region ap-northeast-1 --filters "Name=vpc-id,Values=vpc-0ef535df4de8e80bb" --query 'Subnets[?MapPublicIpOnLaunch==`false`].[SubnetId,Tags]'
```

**發現問題**：Subnet 缺少 ALB 所需的 Kubernetes 標籤：
- 公共 subnet 缺少：`kubernetes.io/role/elb = 1`
- 私有 subnet 缺少：`kubernetes.io/role/internal-elb = 1`

### 解決步驟三：新增 Subnet 標籤

手動新增缺失的標籤：

```bash
# 公共子網路新增標籤
$ aws ec2 create-tags --region ap-northeast-1 --resources subnet-03137f56962d0a148 subnet-0a11863cea526b5a0 subnet-00b4e519a524eb1f3 --tags Key=kubernetes.io/role/elb,Value=1

# 私有子網路新增標籤
$ aws ec2 create-tags --region ap-northeast-1 --resources subnet-00d1d5009e0fd2145 subnet-0dbcd8d5c45e993f5 subnet-087c8cf6e4ba0d2ae --tags Key=kubernetes.io/role/internal-elb,Value=1
```

### 解決結果驗證

標籤新增後，Load Balancer Controller 成功建立 ALB：

```bash
$ kubectl get ingress -n nebuletta-app-ns
NAME           CLASS   HOSTS                   ADDRESS                                                                      PORTS   AGE
ingress-path   alb     eks-lab.nebuletta.com   k8s-nebulettaekslab-4b28eb94af-1752493976.ap-northeast-1.elb.amazonaws.com   80      42m
```

**成功日誌**：
```
{"level":"info","ts":"2025-08-10T07:09:35Z","logger":"controllers.ingress","msg":"created loadBalancer","stackID":"nebuletta-eks-lab","resourceID":"LoadBalancer","arn":"arn:aws:elasticloadbalancing:ap-northeast-1:362395300803:loadbalancer/app/k8s-nebulettaekslab-4b28eb94af/3e2e73703b34a339"}
{"level":"info","ts":"2025-08-10T07:09:35Z","logger":"controllers.ingress","msg":"created listener","stackID":"nebuletta-eks-lab","resourceID":"80","arn":"arn:aws:elasticloadbalancing:ap-northeast-1:362395300803:listener/app/k8s-nebulettaekslab-4b28eb94af/3e2e73703b34a339/d26a631eb694bf64"}
{"level":"info","ts":"2025-08-10T07:09:36Z","logger":"controllers.ingress","msg":"successfully deployed model","ingressGroup":"nebuletta-eks-lab"}
```

**應用程式測試**：
```bash
$ curl k8s-nebulettaekslab-4b28eb94af-1752493976.ap-northeast-1.elb.amazonaws.com:80/beta -H 'Host: eks-lab.nebuletta.com'
[beta] served by: beta-app-deployment-7d7575984d-48jwp

$ curl k8s-nebulettaekslab-4b28eb94af-1752493976.ap-northeast-1.elb.amazonaws.com:80/prod -H 'Host: eks-lab.nebuletta.com'
[prod] served by: prod-app-deployment-68cc5659d9-4w2gm
```

---

## Terraform 模組改善建議

### 1. 使用 EKS Addons 管理系統元件（推薦）

#### 最佳實踐：官方 EKS Addons

經過實際驗證，最佳的解決方案是使用 AWS EKS 的官方 addons 管理系統，而不是手動修補 Kubernetes 資源。這種方法更穩定、更可靠，且完全由 AWS 管理。

**新增 `terraform/modules/eks-lab/cluster/addons.tf`**：
```hcl
# EKS Addons Configuration - 管理基礎系統元件
# 這些 addons 會自動處理 Fargate 相容性問題

# VPC CNI addon - 處理 Pod 網路
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

# Kube-proxy addon - 處理服務網路
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

# CoreDNS addon - 自動配置 Fargate toleration
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

# Metrics Server addon - 支援 HPA 等功能
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

**修改 `terraform/modules/eks-lab/cluster/variables.tf`**，新增版本變數：
```hcl
# EKS Addons 版本變數, 版本可以去 AWS EKS addons 介面看
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

#### 為什麼使用 EKS Addons？

1. **官方支援**：由 AWS 直接維護和管理
2. **自動更新**：可以設定自動更新到相容版本
3. **Fargate 相容性**：自動處理 toleration 和調度需求
4. **一致性**：避免手動修補可能產生的配置漂移
5. **可靠性**：減少跨 provider 操作的複雜度

#### 與舊方法的對比

| 方法 | 手動修補 kubernetes_manifest | EKS Addons |
|------|---------------------------|------------|
| 複雜度 | 高（需要跨 provider 操作）| 低（AWS 原生管理）|
| 可靠性 | 中等（依賴執行順序）| 高（AWS 保證）|
| 維護性 | 困難（需要手動更新）| 簡單（可自動更新）|
| 錯誤處理 | 複雜 | 內建衝突解決 |

### 2. Subnet 標籤修復

修改 `networking` 模組的 `vpc.tf`：

```hcl
# 公共子網路
resource "aws_subnet" "public" {
  count                   = length(local.public_subnet_cidrs)
  vpc_id                  = aws_vpc.this.id
  cidr_block              = local.public_subnet_cidrs[count.index]
  availability_zone       = local.available_azs[count.index]
  map_public_ip_on_launch = true
  
  tags = merge(local.common_tags, {
    Name                     = "${local.prefix}-public-${count.index}"
    "kubernetes.io/role/elb" = "1"  # 新增此標籤
  })
}

# 私有子網路
resource "aws_subnet" "private" {
  count             = length(local.private_subnet_cidrs)
  vpc_id            = aws_vpc.this.id
  cidr_block        = local.private_subnet_cidrs[count.index]
  availability_zone = local.available_azs[count.index]
  
  tags = merge(local.common_tags, {
    Name                              = "${local.prefix}-private-${count.index}"
    "kubernetes.io/role/internal-elb" = "1"  # 新增此標籤
  })
}
```

#### 為什麼標籤值必須是 "1"？

這是 AWS 官方規範的重要細節，許多人會疑惑為什麼不是 `"true"` 或其他值：

**1. AWS 官方文檔要求**
- AWS Load Balancer Controller 官方文檔明確指出標籤值必須為 `"1"`
- 這是 AWS 和 Kubernetes 生態系統中的標準約定

**2. 標籤搜尋機制**
```bash
# AWS Load Balancer Controller 內部會執行類似的搜尋：
aws ec2 describe-subnets --filters "Name=tag:kubernetes.io/role/elb,Values=1"
aws ec2 describe-subnets --filters "Name=tag:kubernetes.io/role/internal-elb,Values=1"
```

**3. 值的比較**
| 標籤值 | 支援狀態 | 說明 |
|-------|----------|------|
| `"1"` | ✅ 官方標準 | AWS 官方文檔要求的值 |
| `"true"` | ❌ 不支援 | 雖然語意上正確，但 Controller 不會識別 |
| `"yes"` | ❌ 不支援 | 同樣不被 Controller 識別 |
| `""` (空值) | ⚠️ 部分支援 | 技術上有效，但不推薦使用 |

**4. 歷史背景**
- 這個慣例源自早期的 Kubernetes 標籤系統
- 在布林標記中，`"1"` 代表 "enabled" 或 "true"
- 維持向後相容性，所以一直沿用至今

**5. 實際驗證**
如果使用錯誤的標籤值，Load Balancer Controller 會報錯：
```
couldn't auto-discover subnets: unable to resolve at least one subnet (0 match VPC and tags: [kubernetes.io/role/elb])
```

**記住**：一定要使用 `"1"`，這不是隨意選擇，而是 AWS 的硬性要求！

---

## 關鍵學習重點

### EKS Fargate 的特殊性

1. **Taint 機制**：Fargate nodes 具有特殊 taint，需要對應的 toleration
2. **系統元件**：CoreDNS 等系統元件可能需要額外設定才能在 Fargate 上運行
3. **DNS 依賴**：整個叢集的網路功能都依賴於 CoreDNS 的正常運作

### ALB Ingress Controller 需求

1. **Subnet 標籤**：必須正確標記 subnet 的用途
   - `kubernetes.io/role/elb`：用於公共 ALB
   - `kubernetes.io/role/internal-elb`：用於內部 ALB
2. **網路連接**：需要能夠存取 AWS API（透過 VPC endpoints 或 NAT Gateway）
3. **IAM 權限**：需要 IRSA 設定正確的 IAM 權限

### 問題診斷技巧

1. **由外而內**：從使用者看到的現象開始追蹤
2. **檢查依賴鏈**：確認每個元件的依賴服務是否正常
3. **分層診斷**：從底層（DNS）到上層（應用程式）逐步檢查
4. **日誌分析**：善用 kubectl logs 找到具體錯誤訊息

### 補充
- 其實在 AWS EKS 實作指南的 [EKS Cluster 建立](../../containerization/orchestration/Kubernetes/zh-tw/35_eks_practice.md#eks-cluster-建立) 章節中，可以從 log 看到以下 log：
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
- 這表示：
  1. EKS Addon 管理：eksctl 使用的是 EKS 的 addon 系統來部署 CoreDNS，而不是單純的 Kubernetes Deployment
  2. Fargate 感知：當使用 eksctl create cluster --fargate 時，eksctl 會：
    - 自動建立預設的 Fargate Profile
    - 自動配置 CoreDNS 使其能在 Fargate 上調度
    - 處理必要的容忍度設定
  3. 與 Terraform 的差異：我們用 Terraform 建立的是：
    - 原生的 EKS cluster
    - 手動建立 Fargate Profile
    - CoreDNS 依然是 EKS 預設部署，但沒有自動 Fargate 調度配置
- 這就是為什麼手動透過 eksctl 操作的時候沒有遇到這個問題，因為被 AWS 先行處理掉了

---

## 預防措施

1. **模組設計**：在基礎設施模組中就考慮 EKS 的特殊需求
2. **自動化測試**：部署後自動驗證 DNS 解析和基本功能
3. **監控告警**：設定 CoreDNS 和 Load Balancer Controller 的健康檢查
4. **文件化**：記錄已知問題和解決方案，避免重複踩坑

---

## 重要注意事項：Terraform Output 與 ALB 就緒時機

### ALB 建立的非同步特性

在部署 applications stack 後，可能會遇到以下狀況：

```bash
# kubectl 顯示 ingress 已有 ADDRESS
$ kubectl get ingress -n nebuletta-app-ns
NAME           CLASS   HOSTS                   ADDRESS
ingress-path   alb     eks-lab.nebuletta.com   k8s-nebulettaekslab-4b28eb94af-1249260893.ap-northeast-1.elb.amazonaws.com

# 但 terraform output 卻顯示空值
$ terramate run --tags dev-eks-applications -- terraform output
ingress_address = ""
ingress_hostname = ""
```

### 原因分析

這是因為：
1. **ALB 建立是非同步過程**：Terraform apply 完成時，ALB 可能還在建立中
2. **Terraform 狀態快照**：output 顯示的是最後一次讀取資源狀態時的值
3. **Kubernetes 即時狀態**：kubectl 顯示的是叢集的即時狀態

### 解決方法

**方法一：刷新 Terraform 狀態（推薦）**
```bash
# 刷新狀態以取得最新的 ingress 資訊
terramate run --tags dev-eks-applications -- terraform refresh

# 再次檢查 output
terramate run --tags dev-eks-applications -- terraform output
```

**方法二：重新執行 apply**
```bash
# 重新 apply 會自動刷新狀態
terramate run --tags dev-eks-applications -- terraform apply -auto-approve

# 檢查 output
terramate run --tags dev-eks-applications -- terraform output
```

**方法三：持續監控**
```bash
# 每 30 秒檢查一次
watch -n 30 "terramate run --tags dev-eks-applications -- terraform output ingress_address"
```

### 時間預期

- **ALB 建立時間**：通常需要 2-5 分鐘
- **DNS 傳播時間**：額外 1-2 分鐘
- **總等待時間**：約 3-7 分鐘

### 驗證 ALB 就緒狀態

```bash
# 方法一：檢查 Kubernetes ingress 狀態
kubectl get ingress -n nebuletta-app-ns

# 方法二：檢查 AWS Load Balancer Controller 日誌
kubectl logs -n kube-system deployment/aws-load-balancer-controller

# 方法三：測試 DNS 解析
dig +short k8s-nebulettaekslab-4b28eb94af-1249260893.ap-northeast-1.elb.amazonaws.com
```

### 提醒

**不用急於除錯**：如果 kubectl 顯示 ingress 有 ADDRESS，但 terraform output 為空，**先等待 ALB 完全就緒**，然後執行 `terraform refresh`。這是正常的非同步行為，不是配置錯誤。

---

## 結論

這次的除錯經驗展示了 EKS Fargate 環境的複雜性，一個看似簡單的 "Ingress 無法建立 ALB" 問題，實際上涉及了：

1. **CoreDNS 調度問題**（Fargate toleration）
2. **DNS 解析失敗**（整個叢集網路功能中斷）
3. **AWS API 存取問題**（STS 連接失敗）
4. **Subnet 標籤缺失**（ALB 無法找到適當的 subnet）

**關鍵**：在 EKS Fargate 環境中，系統元件的正確設定是整個叢集功能的基礎。CoreDNS 作為 DNS 服務的核心，其失效會導致連鎖反應，影響所有需要進行 DNS 查詢的服務。問題的表象往往不是 root cause，需要透過系統性的分析找到真正的根本問題。
