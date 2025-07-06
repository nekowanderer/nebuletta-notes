# EKS Fargate Cluster 架構與機制

[English](../en/15_aws_eks_fargate_cluster.md) | [繁體中文](../zh-tw/15_aws_eks_fargate_cluster.md) | [日本語](../ja/15_aws_eks_fargate_cluster.md) | [回到索引](../README.md)

### 什麼是 EKS Fargate

EKS Fargate 是 Amazon 提供的 serverless 容器運行方式，讓使用者可以在不需要管理 EC2 instance 的情況下運行 Kubernetes Pod。Fargate 會自動為每個 Pod 分配適當的運算資源，並完全接管底層基礎設施的管理。

### 架構圖
```                                                                                                                                                                                                                                                                        
                                                           AWS EKS                                                                       

              Admin EC2                                    Cluster                                     Fargate Computing                 
   +-------------------------------+    +------------------------------------------+      +------------------------------------------+   
   | +--------+                    |    |                                          |      | +-------------+  +---------------------+ |   
   | | eksctl |                    |    |                                          |      | | Master Node |  | Master Replica Node | |   
   | +--------+                    |    | +-------------+      +-----------------+ |      | +-------------+  +---------------------+ |   
   |  |                            |    | | Pod: my-pod |      | Pod: my-pod     | |      +------------------------------------------+   
   |  |--create k8s cluster -------|--->| | ns: app-ns  |      | ns: kube-system | |                                                     
   |  |                            |    | +------|------+      +--------|--------+ |                   Fargate Computing                 
   |  +--create cluster resources -|--->|        |                      |          |      +------------------------------------------+   
   |                               |    |        |                      |          |      |  +-------------+     +-------------+     |   
   +-------------------------------+    +--------|----------------------|----------+      |  | Worker Node |     | Worker Node |     |   
                                                 \                      \                 |  +-------------+     +-------------+     |   
                                                  |                      |                |     +-------------+     +-------------+  |   
                                         +-----------------+    +-----------------+       |   ->| Worker Node |     | Worker Node |  |   
                                         | Fargate Profile |    | Fargate Profile |-------|--/  +-------------+     +-------------+  |   
                                         +-----------------+    +-----------------+       |                                          |   
                                                  |                                       |    +-------------+    +-------------+    |   
                                                  ----------------------------------------|--->| Worker Node |    | Worker Node |    |   
                                                                                          |    +-------------+    +-------------+    |   
                                                                                          +------------------------------------------+   
```

### 核心架構元件

根據架構圖可以看到 EKS Fargate 包含以下關鍵元件：

- **Resource Provider**：左側的 AWS 資源提供者，包含 Admin EC2 和 eksctl 工具
- **Kubernetes Cluster**：中間的 K8s 集群，包含各種 Pod 和 Namespace
- **Fargate Profile**：Fargate 設定檔，定義哪些 Pod 要在 Fargate 上運行
- **Fargate Computing**：實際的 Fargate 運算資源，分為 Master Node 和 Worker Node

### EKS Fargate 的運作機制

```
Admin EC2 → eksctl → Kubernetes Cluster → Fargate Profile → Fargate Computing
```

1. **集群建立**：透過 Admin EC2 上的 eksctl 工具建立 K8s cluster
2. **資源配置**：eksctl 會建立 cluster 所需的各種資源
3. **Pod 部署**：Pod 被部署到指定的 Namespace（如 `app-ns`、`kube-system`）
4. **Profile 配對**：Fargate Profile 會根據設定的規則選取符合條件的 Pod
5. **資源分配**：選中的 Pod 會被分配到 Fargate Computing 資源上執行

### Fargate Profile 的作用

Fargate Profile 是決定哪些 Pod 要在 Fargate 上運行的關鍵配置：

- **Namespace 選擇**：可以指定特定的 Namespace（如圖中的 `app-ns`）
- **Pod 選擇器**：透過 Label Selector 精確控制哪些 Pod 使用 Fargate
- **資源隔離**：確保不同的工作負載在適當的運算環境中執行

### Fargate 與傳統 EC2 的差異

- **無需管理節點**：Fargate 會自動管理底層的運算資源，無需手動配置 EC2 實例
- **按需計費**：只需為實際運行的 Pod 付費，無需為閒置的節點付費
- **自動擴展**：Fargate 會根據 Pod 的需求自動分配適當的運算資源
- **更高的安全性**：每個 Pod 都在獨立的運算環境中運行，提供更好的隔離性

### 使用場景與考量

**適合使用 Fargate 的場景：**
- 不想管理 EC2 實例的複雜性
- 工作負載具有間歇性或不可預測的特性
- 需要快速啟動和關閉的應用程式
- 對安全隔離有較高要求的環境

**注意事項：**
- Fargate 的成本可能比 EC2 更高，特別是對於長時間運行的工作負載
- 某些 K8s 功能可能在 Fargate 上有限制
- 需要適當設計 Fargate Profile 以確保 Pod 正確調度
