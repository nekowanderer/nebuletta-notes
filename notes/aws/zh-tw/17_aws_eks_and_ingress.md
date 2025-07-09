# AWS EKS 與 Ingress 的整合

[English](../en/17_aws_eks_and_ingress.md) | [繁體中文](../zh-tw/17_aws_eks_and_ingress.md) | [日本語](../ja/17_aws_eks_and_ingress.md) | [回到索引](../README.md)

## 架構概覽

```
              Admin EC2                                   Cluster                                                           
  +-------------------------------+    +------------------------------------------------+                                   
  | +--------+                    |    |  +----Service A----+       +---Service B----+  |                                   
  | | eksctl |                    |    |  | +-----+ +-----+ |       |+-----+ +-----+ |  |                                   
  | +--------+                    |    |  | | Pod | | Pod | |-------|| Pod | | Pod | |  |                                   
  |  |                            |    |  | +-----+ +-----+ |   |   |+-----+ +-----+ |  |                                   
  |  |--create k8s cluster ----------->|  +-----------------+   |   +----------------+  |                                   
  |  |                            |    |                        |                       |                                   
  |  +--create cluster resources ----->|            +--------------------+              |                                   
  |                               |    |            |       Ingress      |              |              +---------+          
  | +---------+                   |    |            +--------------------+              |          ----| AWS ALB |          
  | | kubectl |                   |    |                        |                       |    -----/    +---------+          
  | +---------+                   |    |           +------------------------+          -----/               ^               
  |  |                            |    |           | +--------------------+ |    -----/ |                   |               
  |  |--access k8s cluster ----------->|           | | AWS Load Balancer  | |---/       |                   |  able to create               
  |                               |    |           | | Ingress Controller | |           |                   |               
  +-------------------------------+    |           | +--------------------+ |           |              +----------+         
                                       |           | +--------------------+ |           |              | IAM ROLE |         
                                       |           | |   Service Account  | |           |              +----------+         
                                       |           | +--------------------+ |           |                   |               
                                       |           +------------------------+           |                   |               
                                       |                        |                       |                   |               
                                       |         +-----------------------------+        |        +-----------------------+  
                                       |         | OpenID Connect Provider URL |-----------------| IAM Identity Provider |  
                                       |         +-----------------------------+        |        +-----------------------+  
                                       +------------------------------------------------+                                   
```

## 架構組成

### 管理端 (Admin EC2)

#### eksctl 工具
- **用途**: 建立和管理 EKS 叢集
- **主要功能**:
  - 建立 Kubernetes 叢集
  - 設定叢集資源 (IAM 角色、安全群組等)
  - 管理節點群組

#### kubectl 工具
- **用途**: Kubernetes 叢集管理工具
- **主要功能**:
  - 存取和管理叢集內的資源
  - 部署應用程式
  - 監控叢集狀態

### EKS 叢集內部

#### 應用程式服務層
- **Service A/B**: 包含多個 Pod 的服務
- **負載平衡**: 服務內部使用 Kubernetes 的負載平衡機制

#### Ingress 資源
- **作用**: 定義外部流量如何進入叢集
- **功能**: 路由規則設定、SSL/TLS 終止、主機名稱和路徑基礎的路由

## 權限管理架構

### AWS EKS 與本地 K8s 的最大差異

#### OpenID Connect Provider URL
- EKS 自動產生 OIDC Provider URL
- 讓 K8s Cluster 具備可識別的「身分」，使 K8s 裡的元件可以向 AWS 申請權限（IAM Role）

#### IAM Identity Provider + Role
1. **建立 IAM Identity Provider**: 綁定 EKS 的 OIDC Provider
2. **建立 IAM Role**: 授權這個 Role 可以建立 Load Balancer（ALB）
3. **綁定 Service Account**: 讓 Ingress Controller 可以 assume 這個 Role，取得建立 ALB 的權限

## Ingress Controller 運作機制

### AWS Load Balancer Controller
- **部署方式**: K8s 內部署 AWS Load Balancer Ingress Controller
- **監控對象**: 這個 Controller 會監控 K8s 的 Ingress 資源
- **自動化**: 當建立/更新 Ingress 時，Controller 會自動在 AWS 建立 ALB

### Service Account 與 IAM Role 串接
- Ingress Controller 需要權限操作 AWS 資源
- 透過 Service Account 結合 IAM Role
- 提供安全的權限管理機制

## 流量流程

### 外部流量進入流程
1. **使用者請求** → AWS ALB
2. **ALB 路由** → 依照 Ingress 規則 (Host/Path) 導向對應的 K8s Service
3. **Service 負載平衡** → 目標 Pod

### ALB 特性
- **層級**: L7 層 (應用層)
- **功能**: 可以做 Host/Path 分流
- **優勢**: 比 NLB 多了更多彈性

## Service Account 詳細說明

### 什麼是 Service Account？
- **定義**: Kubernetes 的「機器用帳號」
- **用途**: 讓 Pod 裡面的應用程式擁有自己的「身分」
- **預設**: 每個 Pod 都有 default Service Account

### 主要用途
- **控制 Pod 存取**: 管理 Pod 可以存取哪些 K8s API
- **權限分離**: 為了安全性，把不同權限分開管理

### 與 AWS IAM 的關係

#### 綁定機制
- Service Account 可以跟 AWS IAM Role 綁定
- Pod 使用特定 SA 啟動後，自動擁有對應 IAM Role 的權限
- **優勢**: 不需要在程式或環境變數中寫入 Access Key/Secret Key

#### 實際綁定流程
1. **建立 IAM Role**: 信任 EKS OIDC Provider
2. **建立 Service Account**: 在 metadata 加上 annotation 綁定 IAM Role ARN
3. **自動頒發憑證**: AWS 自動頒發 temporary credentials 給 Pod

#### 範例配置
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: alb-ingress-controller
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/ALBIngressControllerRole
```

### 關鍵概念總結
- **Service Account** = K8s 裡「給機器用」的帳號
- **IAM Role** = AWS 服務授權的角色
- **兩者結合** = K8s 上的應用可以安全取得 AWS 資源操作權限

## AWS Load Balancer Controller 運作詳解

### Controller 本身也是 Pod
- **部署方式**: K8s Deployment，包含一組或多組 Pod
- **權限需求**: 需要操作 AWS ALB 資源的權限
- **解決方案**: 綁定特製的 Service Account 和 IAM Role

### 完整運作流程
1. **監控變動**: Controller 偵測 Ingress 資源變動
2. **建立資源**: 透過 IAM Role 權限，自動在 AWS 上建立/更新 ALB
3. **流量路由**: ALB 接收流量，根據 Ingress 規則分流至不同 Service

## 總結

### 與本地 K8s 的主要差異
- **權限串接**: 所有跨服務資源 (如 ALB) 都必須先完成 IAM 設定
- **自動化程度**: K8s Ingress Controller 可以自動建立與管理 AWS 的 ALB
- **優勢**: 享受 ALB 的高可用與維運優勢

### 核心價值
- **安全性**: 透過 IAM 角色管理權限，避免將憑證寫死
- **自動化**: 減少手動配置工作
- **整合性**: 與 AWS 服務深度整合
