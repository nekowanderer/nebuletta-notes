# Kubernetes PV、PVC、StorageClass 筆記

[English](../en/21_k8s_pv_pvc_sc.md) | [繁體中文](../zh-tw/21_k8s_pv_pvc_sc.md) | [日本語](../ja/21_k8s_pv_pvc_sc.md) | [回到索引](../README.md)

<img src="../images/21_pv_pvc_sc.jpg" width="700">

## 基本概念

- **Pod**：Kubernetes 中運行應用程式的最小單位，內含 Container（容器），可掛載 Volume（儲存卷）。
- **PVC（Persistent Volume Claim）**：Pod 透過 PVC 來請求儲存資源，相當於「我要一塊儲存空間」的申請單。
- **PV（Persistent Volume）**：實際提供儲存空間的資源，可以是靜態（static）或動態（dynamic）建立。

## StorageClass

- **用途**：定義儲存資源的類型、參數與動態配置方式。
- **常見情境**：當 PVC 指定 StorageClass 時，Kubernetes 會根據該 StorageClass 的設定，自動建立對應的 PV。
- **範例**：可以指定不同的 StorageClass 來選擇 SSD、HDD、雲端儲存等不同儲存後端。
- **常見欄位**：
  - `provisioner`：指定由哪個外掛（如 nfs、csi）來建立儲存。
  - `parameters`：細部設定（如儲存大小、I/O 類型等）。

## Manual create（手動建立 PV）

- **用途**：由管理員預先建立好 PV，指定儲存來源（如本地磁碟、NFS 伺服器等）。
- **流程**：
  1. 管理員手動撰寫 PV 的 YAML 檔案，並套用到叢集。
  2. 使用者建立 PVC，Kubernetes 會自動將 PVC 綁定到符合條件的 PV。
- **適用情境**：需要精確控制儲存資源來源，或有特殊需求時。

## Provisioner（Dynamic Provisioning, 動態配置器）

- **用途**：負責根據 StorageClass 的設定，自動建立 PV。
- **常見 Provisioner**：
  - `kubernetes.io/aws-ebs`：AWS EBS
  - `kubernetes.io/gce-pd`：GCP Persistent Disk
  - `nfs.csi.k8s.io`：NFS
  - `csi.azure.com`：Azure Files
- **運作方式**：
  1. 使用者建立 PVC 並指定 StorageClass。
  2. Provisioner 根據 StorageClass 設定，自動建立 PV 並連接到實體儲存。
- **優點**：自動化、彈性高，適合雲端或大規模動態需求。

## 支援的儲存後端

- **本地儲存（Local）**：hostPath
- **雲端儲存**：
  - AWS EFS
  - GCP Filestore
  - Azure Files
- **其他協定**：
  - NFS
  - iSCSI
  - CSI（Container Storage Interface）

## 流程總結

1. Pod 需要儲存空間，提出 PVC。
2. PVC 會去找符合條件的 PV（靜態或動態）。
3. 若為動態，會根據 StorageClass 由 Provisioner 自動建立 PV。
4. PV 會連接到實際的儲存後端（本地或雲端）。
5. Pod 綁定 PVC，最終掛載到 PV 提供的儲存空間。

## 比較白話的比喻

角色
  - Pod（承租人）：想找地方住（存放資料），但不需要知道房東是誰、地址在哪，只在意「租約」條件。
  - PVC（租約）：承租人根據需求（空間大小、存取權限等）寫好的租約申請單。只要契約內容符合，之後直接用這個契約去進房間。
  - PV（房間/倉庫/實體資源）：真正的物理空間（儲存），背後可以是各種硬碟、NFS、雲端等。
  - Kubernetes（仲介/物業管理員）：負責根據租約（PVC）媒合適合的房間（PV），配對完成後負責日常管理和維護。

運作方式
  - Pod 只要提出租約申請（PVC），描述想要什麼樣的空間即可。
  - Kubernetes 會幫你找/蓋一個適合的房間（PV），自動媒合給你。
  - Pod 永遠都是透過租約（PVC）來開門進房間，不會直接知道房間的詳細資料，這樣即使之後房間換人、搬遷，Pod 也不需要更改存取方式。
  - 管理員（Kubernetes）讓資源調度變得很靈活，也方便資源回收、搬家或擴充。

Kubernetes 的 PVC 租屋流程，就像日本租房子一樣，仲介（Kubernetes）打理一切，房客（Pod）只要管租約（PVC），房東（PV 背後的儲存系統）是誰並不是房客需要關心的事情。
