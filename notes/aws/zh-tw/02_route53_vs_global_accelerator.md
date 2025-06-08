# Route 53 與 Global Accelerator 的性質與差異比較

[English](../en/02_route53_vs_global_accelerator.md) | [繁體中文](02_route53_vs_global_accelerator.md) | [日本語](../ja/02_route53_vs_global_accelerator.md) | [回到索引](../README.md)


## 背景

新專案採用 Keycloak，並且會區隔出 admin/public node，其中：
- Admin node：僅限於公司內網，無法透過 Internet 直接存取
- Public node：Internet 可直接存取，但通常是通過公司的前端系統間接存取
- Keycloak 必須實現跨 region 的架構設計


## 解決方案

#### ELB
- ELB 是 Region 級資源
- 不論是 Classic ELB、Application Load Balancer (ALB)，或是 Network Load Balancer (NLB)，它們都只能存在於單一 AWS Region。這代表：
- 無法用同一個 ELB 同時分流到東京（ap-northeast-1）和新加坡（ap-southeast-1）兩個 Region 的 EC2。
- 每個 Region 需要建立自己的 ELB。

#### Route 53
- 可以在 Route 53 裡定義多個 Region 的 ELB DNS
- 利用 Route 53 的 Latency-based routing 或 Geolocation routing
- 加上健康檢查（Health Check）自動 Failover

#### Global Accelerator
- 是一個全球性的服務，讓你可以有一個 anycast IP
- 自動將流量導向最近、健康的 ELB
- 更適合需要低延遲與高可用性的場景


## 關於 Route 53

Route 53 是一個屬於 全球性（Global）DNS 服務，它不是 Region-bound 的元件。以下是它的性質與分類方式：

| 分類角度            | 說明                                   |
| ----------------- | ------------------------------------ |
| **服務類型**        | DNS 服務（Domain Name System）           |
| **作用層級**        | Global（跨 Region，單一服務覆蓋全球）            |
| **用途分類**        | 網域名稱解析、流量導向、健康檢查、高可用性、容錯等            |
| **運作模式**        | 基於 Authoritative DNS（授權式 DNS）        |
| **非 Region 綁定**  | 建立 Hosted Zone、記錄集不屬於任何特定 AWS Region |

#### 提供哪些功能？
- DNS 託管服務（Hosted Zone）
  - 幫你託管 domain 的 DNS 記錄（A, AAAA, CNAME, MX, TXT 等）。
- 流量路由策略
  - Simple Routing（單一 IP/服務）
  - Latency-based Routing（依照延遲最近）
  - Geolocation Routing（根據使用者位置）
  - Weighted Routing（加權分流）
  - Failover Routing（主備切換）
- 健康檢查（Health Check）
  - 可針對 ALB、EC2 或任何 endpoint 設定定期檢查，自動剔除 unhealthy target。
- 整合 AWS Global Accelerator 或 ELB
  - 提供穩定的全球網路接入點。

#### 重點
| 特性                  | 說明                                   |
| ------------------- | ------------------------------------ |
| **Global 資源**       | 不屬於任何單一 AWS Region                   |
| **支援跨 Region 分流**   | 可搭配 health check 實作跨 Region failover |
| **與 ELB 整合**        | 可直接解析 ELB 的 DNS 名稱                   |
| **與 CloudFront 搭配** | 可作為 CDN 的來源導流層                       |



## 關於 Global Accelerator

Amazon Global Accelerator 跟 Route 53 雖然都屬於 Global 級別的服務，但它們性質不同、用途也有明顯差異。

| 分類角度            | 說明                                             |
| --------------- | ---------------------------------------------- |
| **服務類型**        | 全球應用加速服務（Anycast IP + 智慧流量導向）                  |
| **作用層級**        | Global（全球級網路層）                                 |
| **用途分類**        | 流量最佳化、高可用性、自動 Failover、提升跨區網速                   |
| **運作層級**        | TCP/UDP 層，針對 L4 網路封包導流                         |
| **非 Region 綁定** | 本身是 global 服務，但 target 會連到特定 Region 的資源（如 ALB） |

#### 提供哪些功能?
- 提供固定的 Anycast IP
  - 你部署 Global Accelerator 後會拿到固定的 IPv4 公網 IP
  - 使用者來自世界各地，會自動導向離其最近的 AWS 邊緣節點
- 流量導向最近的 Region
  - 透過 AWS 全球骨幹網路（比 public Internet 快且穩定）
  - 根據使用者距離、健康狀態、優先順序導向最適合的 backend
- 支援健康檢查與自動 Failover
  - 若某一個 Region 發生問題，流量會自動切換到另一個健康的目標
- 支援 ALB、NLB、EC2 作為 backend
  - Target 可以是多個不同 Region 的 Load Balancer

#### 適合的場景
- 需要低延遲、即時導向的應用（如線上遊戲、語音聊天、金融服務）
- 全球用戶存取同一服務，期望穩定且快速的網路體驗
- 想要一組固定 IP，不暴露 Load Balancer 的 DNS 名稱
- 想要在多個 Region 部署應用，並根據延遲自動導流


## 關於費用

Global Accelerator 明顯比 Route 53 貴很多，通常只適合對延遲很敏感、或是有全球使用者的高可用服務。

#### 💰 Route 53 成本概念（便宜）
| 項目          | 成本（USD）                 |
| ----------- | ----------------------- |
| Hosted Zone | \$0.50／每月／每個 domain     |
| DNS 查詢      | 約 \$0.40～\$0.60／每百萬次查詢  |
| 健康檢查        | 約 \$0.50／每個 endpoint／每月 |
- 就算是中型網站，Route 53 一整個月下來也可能只花 幾美元。

#### 💰 Global Accelerator 成本概念（昂貴）
| 項目                   | 成本（USD）                             |
| -------------------- | ----------------------------------- |
| 基本費用（每個 Accelerator） | \$0.025／小時 ≒ \$18／月                 |
| 加速 IP（每個）            | \$0.025／小時 ≒ \$18／月 × 2 個 IP = \$36 |
| 流量費（進出流量）            | 視來源地區與目的地區而定（每 GB 約 \$0.015～\$0.12） |
- 用 GA 載一個全球性的 Web application，一個月開著不跑流量也大概 $50～$70 美元，跑流量會再額外增加（每 GB 計費）


## 對比

| 特性          | Route 53                | Global Accelerator     |
| ----------- | ----------------------- | ---------------------- |
| **主要功能**    | DNS 解決與流量導向             | TCP/UDP 流量加速與全球導流      |
| **適合流量類型**  | 各類服務（Web, Email, API）   | 延遲敏感、長連線、遊戲伺服器、即時互動類服務 |
| **IP 穩定性**  | 依賴 DNS 記錄（不固定）          | 提供固定的 Anycast IP（IPv4） |
| **切換時間**    | 可能有 DNS TTL 延遲（數十秒～數分鐘） | 幾乎即時（秒級）               |
| **健康檢查與容錯** | 有，但依賴 DNS 層級            | 內建自動健康檢查與實時 Failover   |
| **導流邏輯**    | DNS 記錄導向、可設定多策略         | 自動導向最近、最健康的 endpoint   |
| **使用成本**    | 相對便宜（依據 DNS 查詢次數與監控計費）  | 相對昂貴（依流量、加速 IP 等項目計費）  |
| **類比** | 智慧 DNS 導流 | 跨國網路 VIP 專線 + 導流器 |


## 參考文獻

- [Amazon Route 53 定價](https://aws.amazon.com/tw/route53/pricing/)
- [AWS Global Accelerator 定價](https://aws.amazon.com/tw/global-accelerator/pricing/)
