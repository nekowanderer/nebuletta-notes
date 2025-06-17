# Kubernetes 架構總覽

[English](../en/03_k8s_architecture_overview.md) | [繁體中文](../zh-tw/03_k8s_architecture_overview.md) | [日本語](../ja/03_k8s_architecture_overview.md) | [回到索引](../README.md)

---

## Kubernetes 架構圖

```
+--------------------------- Cluster ----------------------------+
|   +-------------------+                +-------------------+   |
|   |    Master Node    |                |    Worker Node    |   |
|   |  +-------------+  |                |  +-------------+  |   |
|   |  | API Server  |  |                |  | Kubelet     |  |   |
|   |  |             |  |                |  | Service     |  |   |
|   |  +-------------+  |                |  +-------------+  |   |
|   |  | Cluster     |  |    control     |  | Kube Proxy  |  |   |
|   |  | Controller  |  | <============> |  | Service     |  |   |
|   |  | Manager     |  |  communicate   |  +-------------+  |   |
|   |  +-------------+  |                |  | Container   |  |   |
|   |  | Scheduler   |  |                |  | Runtime     |  |   |
|   |  +-------------+  |                |  +-------------+  |   |
|   |  | Cluster     |  |                |                   |   |
|   |  | State       |  |                |                   |   |
|   |  | Store       |  |                |                   |   |
|   |  +-------------+  |                |                   |   |
|   +-------------------+                +-------------------+   |
+----------------------------------------------------------------+
```

---

## About Master Node（主節點）

Master Node 是 Kubernetes 叢集的「大腦」，負責整個叢集的管理、調度與狀態維護。主要元件如下：

- **API Server**：
  - 叢集的溝通樞紐，所有指令（CLI、YAML）都會先到這裡。
  - 提供 REST API，負責驗證、授權、資料驗證與調度。
- **Cluster Controller Manager**：
  - 管理各種控制器（如節點、複本、終端點等），確保叢集狀態符合預期。
- **Scheduler**：
  - 負責將新建立的 Pod 分配到合適的 Worker Node 上。
- **Cluster State Store（etcd）**：
  - 分散式 key-value 資料庫，儲存整個叢集的狀態資料。
- **Master Replicate Node**：
  - 提供高可用性，當主節點故障時可自動切換。

---

## About Worker Node（工作節點）

Worker Node 是實際執行應用程式容器（Pod）的地方。主要元件如下：

- **Kubelet Service**：
  - 負責與 Master 溝通，並管理本機的 Pod 生命週期。
- **Kube Proxy Service**：
  - 負責網路代理與負載平衡，讓服務可以跨節點存取。
- **Container Runtime**：
  - 實際執行容器的軟體（如 Docker、Podman）。

---

## Pod 與 Container Registry 說明

Kubernetes 中，Pod 是最小的可部署單位，一個 Pod 裡面可以包含一個或多個 Container（容器）。每個 Container 都是根據映像檔（Image）啟動的，而這些映像檔會儲存在 Container Registry（容器登錄中心），可以是本地端（Local）或雲端（如 DockerHub、AWS ECR、GCP GAR、Azure ACR 等）。

當你在 K8S 叢集裡部署 Pod 時，Kubernetes 會自動從指定的 Registry 拉取對應的映像檔，然後在 Worker Node 上啟動對應的 Container。

### Pod 與 Container Registry 關係圖

```
            +--------- Cluster ---------+
            |                           |
            |   +-------------------+   |
            |   |        Pod        |   |
            |   |  +-------------+  |   |
            |   |  | Container   |  |   |
            |   |  | +---------+ |  |   |
            |   |  | | Image   | |  |   |
            |   |  | +---------+ |  |   |
            |   |  +------|------+  |   |
            |   +---------|---------+   |
            |             |             |
            +-------------|-------------+
                          | 拉取映像檔 (pull image)
                          v
+---------------- Container Registry ---------------+
| Local | DockerHub | AWS ECR | GCP GAR | Azure ACR |
+---------------------------------------------------+
```
- **Pod**：K8S 中最小的運作單位，通常對應一個應用服務。
- **Container**：在 Pod 裡執行的實體容器，根據映像檔啟動。
- **Image**：應用程式的執行環境快照，包含程式碼、相依套件等。
- **Container Registry**：集中管理映像檔的地方，可分為本地端或雲端。
- **自動拉取**：K8S 會根據 Pod 設定，自動從 Registry 拉取映像檔。

---

## Pod、Node 與 Resource Provider 關係

Kubernetes 叢集中的 Pod 會被分配到不同的 Worker Node 上執行，而這些 Worker Node 實際上就是運行在各種不同的運算資源（Resource Provider）上，例如本地機器、AWS EC2、GCP Compute Engine、Azure VM 等。

K8S 透過這種設計，讓你可以彈性選擇底層的運算平台，無論是地端還是雲端。

```
            +---------- Kubernetes Cluster ----------+
            |                                        |
            |   +-----+      +-------------------+   |
            |   | Pod |----> |                   |   |
            |   +-----+      |    Worker Node    |   |
            |   +-----+      |                   |   |
            |   | Pod |----> |                   |   |
            |   +-----+      +-------------------+   |
            |   +-----+      +-------------------+   |
            |   | Pod |----> |                   |   |
            |   +-----+      |    Worker Node    |   |
            |                |                   |   |
            |                +---------|---------+   |
            +--------------------------|-------------+
                                       |
                                       v
       +--------- Computing Resource Provider -----------+
       | Local | AWS EC2 | GCP Compute Engine | Azure VM |
       +-------------------------------------------------+
```
- **Worker Node**：K8S 叢集中的運算節點，實際執行 Pod。
- **Resource Provider**：提供運算資源的平台，可以是本地伺服器，也可以是各大雲端服務商的 VM。
- **彈性部署**：K8S 讓你可以同時管理多種不同來源的運算資源，實現混合雲或多雲部署。

---

## 補充說明

- **高可用性設計**：
  - Master Node 通常會有多個複本（Master Replicate Node），避免單點故障。
- **溝通方式**：
  - 使用者可透過 CLI 或 YAML 檔案與 API Server 互動。
  - Master Node 與 Worker Node 之間持續溝通，確保叢集狀態一致。
- **彈性擴展**：
  - Worker Node 可隨時擴增或縮減，方便應對不同的工作負載。
- **支援多種容器執行環境**：
  - 如 Docker、Podman 等。
- **Kubernetes 架構設計理念**：
  - 分層、模組化，讓管理與維護更容易。

---

