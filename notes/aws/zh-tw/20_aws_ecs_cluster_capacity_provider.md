# AWS ECS Cluster Capacity Provider

[English](../en/20_aws_ecs_cluster_capacity_provider.md) | [繁體中文](../zh-tw/20_aws_ecs_cluster_capacity_provider.md) | [日本語](../ja/20_aws_ecs_cluster_capacity_provider.md) | [回到索引](../README.md)
  
## Capacity Provider 是什麼？
  
AWS ECS Capacity Provider 是一種**自動化資源管理機制**，用來控制 ECS Cluster 中運算資源的配置與擴展。它決定了**容器要在哪種基礎設施上運行**，以及**如何自動調整資源規模**。
  
---
  
## 主要類型
  
### 1. EC2 Capacity Provider
- **基於 EC2 執行個體**
- 提供完整的作業系統控制權
- 適合需要特定配置或持久化需求的應用
  
### 2. Fargate Capacity Provider
- **無伺服器容器服務**
- AWS 完全管理基礎設施
- 按需付費，不需要管理 EC2 執行個體
  
### 3. Fargate Spot Capacity Provider
- **基於 Fargate Spot 執行個體**
- 成本較低，但可能被中斷
- 適合容錯性高的工作負載
  
---
  
## 核心功能
  
### 自動擴展（Auto Scaling）
```
目標容量 ──→ [Capacity Provider] ──→ 實際資源配置
     ↑                                    ↓
   監控指標 ←─────── [CloudWatch] ←─────── 資源使用狀況
```
  
### 資源分配策略

#### REPLICA 策略
- **均勻分散任務**到所有 Capacity Provider
- 適合高可用性需求，分散風險

#### BINPACK 策略
- **優先填滿現有資源**，再開啟新資源
- 就像打包行李箱一樣，先把現有空間填滿

### BINPACK 詳細說明

#### 運作原理
假設您有以下 ECS Cluster 配置：
```
EC2 執行個體規格：4 vCPU, 8GB RAM

目前狀況：
Instance-1: 已使用 2 vCPU, 4GB RAM (剩餘 2 vCPU, 4GB)
Instance-2: 已使用 1 vCPU, 2GB RAM (剩餘 3 vCPU, 6GB)
```

當有新任務需要 `1 vCPU, 2GB RAM` 時：

**BINPACK 會選擇**：
- 選擇 Instance-1（使用率較高的機器）
- 結果：Instance-1 使用率提升到 75%

**REPLICA 會選擇**：
- 選擇 Instance-2（平均分散負載）
- 結果：兩台機器使用率都維持中等

#### 實際案例比較
```
場景：10個任務需要部署

BINPACK 結果：
├── Instance-1: 100% 使用率 (8個任務) ✓ 滿載
├── Instance-2: 100% 使用率 (2個任務) ✓ 滿載  
└── Instance-3: 0% 使用率 → 可關閉節省成本

REPLICA 結果：
├── Instance-1: 33% 使用率 (3-4個任務)
├── Instance-2: 33% 使用率 (3-4個任務)
└── Instance-3: 33% 使用率 (2-3個任務) → 都需保持運行
```

#### BINPACK 優點
- **節省成本**：閒置機器可以關閉
- **提升效率**：資源整合度高
- **適合彈性工作負載**：配合 Auto Scaling 使用
- **Spot 執行個體友善**：減少中斷影響範圍

#### BINPACK 注意事項
- **單點故障風險**：熱點機器故障影響較大
- **資源競爭**：可能導致效能瓶頸
- **不適合關鍵服務**：高可用性需求的應用要謹慎使用

#### 使用建議
- **適用場景**：開發測試環境、成本敏感應用、批次處理工作
- **不適用場景**：生產關鍵服務、需要高可用性的應用
  
---
  
## 設定要素
  
### 基本設定
| 項目 | 說明 |
|------|------|
| **Provider Name** | Capacity Provider 識別名稱 |
| **Auto Scaling Group** | 關聯的 EC2 Auto Scaling Group（僅 EC2 類型） |
| **Managed Scaling** | 是否啟用自動擴展 |
| **Target Capacity** | 目標容量百分比（通常設 100%） |
  
### 進階設定
```bash
# 範例：建立 EC2 Capacity Provider
$ aws ecs create-capacity-provider \
    --name my-ec2-capacity-provider \
    --auto-scaling-group-provider \
        autoScalingGroupArn=arn:aws:autoscaling:region:account:autoScalingGroup \
        managedScaling=enabled,status=ENABLED,targetCapacity=100 \
        managedTerminationProtection=ENABLED
```
  
---
  
## 實際運作流程
  
### 1. 任務請求階段
```
ECS Service/Task → Cluster → Capacity Provider → 資源配置
```
  
### 2. 資源不足時
```
需求增加 → 偵測容量不足 → 觸發擴展 → 新增運算資源 → 部署容器
```
  
### 3. 需求降低時
```
負載減少 → 偵測過剩容量 → 觸發縮減 → 回收運算資源 → 節省成本
```
  
---
  
## 混合策略配置
  
### 多 Provider 組合
```yaml
# 範例：混合使用策略
Cluster:
  CapacityProviders:
    - EC2-Provider:      weight: 2
    - Fargate-Provider:  weight: 1
    - FargateSpot-Provider: weight: 1
```
  
### 使用場景建議
- **生產環境穩定服務**：EC2 + Fargate
- **開發測試環境**：Fargate Spot
- **混合工作負載**：三種 Provider 組合使用
  
---
  
## 成本最佳化策略
  
### 1. Spot 執行個體整合
- 利用 Fargate Spot 降低成本達 70%
- 設定適當的容錯機制
  
### 2. 智慧資源配置
- 根據應用特性選擇適合的 Provider
- 設定合理的擴展策略參數
  
### 3. 監控與調整
```bash
# 監控 Capacity Provider 使用狀況
$ aws ecs describe-capacity-providers \
    --capacity-providers my-capacity-provider
```
  
---
  
## 注意事項
  
### 設定限制
- EC2 Capacity Provider 必須關聯有效的 Auto Scaling Group
- Managed Scaling 啟用後，不建議手動調整 ASG
  
### 最佳實務
- 設定適當的 `targetCapacity` 值（建議 100%）
- 啟用 `managedTerminationProtection` 避免意外終止
- 定期檢查 CloudWatch 指標進行調整
  
### 故障排除
```bash
# 檢查 Capacity Provider 狀態
$ aws ecs describe-clusters --cluster my-cluster \
    --include CAPACITY_PROVIDERS
```
  
---
  
## 總結
  
Capacity Provider 是 ECS 自動化運維的核心機制，透過它可以：
  
1. **簡化資源管理**：自動處理容量規劃
2. **最佳化成本**：根據需求動態調整資源
3. **提升可靠性**：多樣化的基礎設施選擇
4. **增強彈性**：支援混合雲端架構
  
選擇合適的 Capacity Provider 策略，能大幅提升 ECS 叢集的效率與經濟性。
  