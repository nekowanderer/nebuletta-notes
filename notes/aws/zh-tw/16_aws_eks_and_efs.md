# AWS EKS 與 EFS 永久儲存整合

[English](../en/16_aws_eks_and_efs.md) | [繁體中文](../zh-tw/16_aws_eks_and_efs.md) | [日本語](../ja/16_aws_eks_and_efs.md) | [回到索引](../README.md)

## 概述

在 Kubernetes 環境中，Pod 的生命週期是短暫的，當 Pod 重新啟動或遷移時，容器內的資料會遺失。為了實現資料的永久儲存，AWS EKS 提供了與 EFS (Elastic File System) 的整合方案，透過 CSI (Container Storage Interface) 驅動程式來管理儲存資源。

## 架構圖解

```                                                          
              Admin EC2                                   Cluster                                                                     
  +-------------------------------+    +------------------------------------------------+                                             
  | +--------+                    |    |                                                |                                             
  | | eksctl |                    |    |                                 PV             |                                             
  | +--------+                    |    |                             +--------+     +-------+      +----------+      +-------------+  
  |  |                            |    |         Pod                 | static |---->|  csi  |----->| Security |----->| AWS EFS     |  
  |  |--create k8s cluster -------|--->|  +---------------+          +--------+     +-------+      | Group    |      | File System |  
  |  |                            |    |  | +-----------+ |               |             |          +----------+      +-------------+  
  |  +--create cluster resources -|--->|  | | container | |               |             |                                             
  |                               |    |  | +-----------+ |       +--------------+      |                                             
  | +---------+                   |    |  |   |           |       | StorageClass |      |                                             
  | | kubectl |                   |    |  |   |-image     |       +--------------+      |                                             
  | +---------+                   |    |  |   |           |               |             |                                             
  |  |                            |    |  |   +-Volume ---|-----\         |             |                                             
  |  +--access k8s cluster -------|--->|  +---------------+      ---\  +-----+          |                                             
  |                               |    |                             ->| PVC |          |                                             
  +-------------------------------+    |                               +-----+          |                                             
                                       |                                                |                                             
                                       |                                                |                                             
                                       +------------------------------------------------+                                                                
```

## 核心元件說明

#### CSI (Container Storage Interface) 驅動程式

CSI 是 Kubernetes 與儲存系統之間的標準化介面，AWS EFS CSI Driver 是專門為 EKS 設計的驅動程式，負責：

- **儲存資源管理**：動態建立和管理 EFS 檔案系統
- **掛載操作**：將 EFS 檔案系統掛載到 Pod 中
- **權限控制**：管理 EFS 的存取權限和安全群組
- **生命週期管理**：處理儲存資源的建立、刪除和更新

#### StorageClass

StorageClass 定義了動態儲存配置的模板：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-xxxxx
  directoryPerms: "700"
```

**重要參數說明：**
- `provisioner`: 指定使用 EFS CSI 驅動程式
- `provisioningMode`: 配置模式（efs-ap 表示使用 Access Point）
- `fileSystemId`: EFS 檔案系統 ID
- `directoryPerms`: 目錄權限設定

#### PersistentVolumeClaim (PVC)

PVC 是 Pod 請求儲存資源的聲明：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```

**關鍵特性：**
- `ReadWriteMany`: 支援多個 Pod 同時讀寫
- `storageClassName`: 指定使用的 StorageClass
- `storage`: 請求的儲存容量（EFS 實際上是按使用量計費）

#### PersistentVolume (PV)

PV 是實際的儲存資源，由 CSI 驅動程式動態建立：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-xxxxx::fsap-xxxxx
```

## CSI 驅動程式的優勢

#### 標準化介面
- 遵循 Kubernetes CSI 標準
- 與其他儲存系統相容
- 簡化儲存管理流程

#### 動態配置
- 按需建立儲存資源
- 自動管理 PV 生命週期
- 減少手動配置工作

#### 多租戶支援
- 透過 Access Point 實現隔離
- 支援不同的權限設定
- 提高安全性

#### 高可用性
- 跨可用區域的資料複製
- 自動故障轉移
- 99.99% 的可用性保證

## 安全考量

#### 網路安全
- 使用 VPC 端點減少網路延遲
- 配置適當的安全群組規則
- 限制 EFS 的網路存取

#### 存取控制
- 使用 IAM 角色進行認證
- 配置 EFS Access Point 權限
- 實施最小權限原則

#### 資料加密
- 啟用傳輸中加密 (TLS)
- 啟用靜態加密 (KMS)
- 定期輪換加密金鑰

## 最佳實踐

#### 效能優化
- 使用 EFS 效能模式 (General Purpose 或 Max I/O)
- 配置適當的吞吐量模式
- 監控 I/O 效能指標

#### 成本管理
- 選擇合適的儲存類別
- 監控儲存使用量
- 實施生命週期政策

#### 備份策略
- 定期建立 EFS 快照
- 測試災難復原程序
- 建立資料保留政策

## 故障排除

#### 常見問題

1. **PVC 無法綁定**
- 檢查 StorageClass 配置
- 驗證 EFS 檔案系統狀態
- 確認 CSI 驅動程式運作正常

2. **Pod 無法掛載 EFS**
- 檢查安全群組規則
- 驗證網路連線
- 確認 IAM 權限設定

3. **效能問題**
- 監控 EFS 效能指標
- 調整應用程式 I/O 模式
- 考慮使用 EFS 效能模式

## 總結

透過 AWS EFS CSI Driver，EKS 叢集可以輕鬆實現永久資料儲存。CSI 驅動程式提供了標準化的介面，簡化了儲存管理流程，同時確保了資料的可靠性和安全性。正確配置和使用 EFS 可以為容器化應用程式提供高效能、高可用性的儲存解決方案。
