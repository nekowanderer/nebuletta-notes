# AWS Route Table 設計討論

[English](../en/04_aws_route_table_design.md) | [繁體中文](04_aws_route_table_design.md) | [日本語](../ja/04_aws_route_table_design.md) | [回到索引](../README.md)

![VPC 資源關係圖](../images/vpc_resource_map.jpg)

## 問題一：為什麼 Public Route Table 只有一個，而 Private Route Table 卻會有多個（by AZ）？



#### Public Route Table 只有一個的原因：
- 公開子網路（Public Subnet）的主要目的是提供網際網路的存取
- 所有公開子網路都需要相同的路由規則：透過網際網路閘道（Internet Gateway）連接到網際網路
- 因此，一個公開路由表就足夠處理所有公開子網路的流量

#### Private Route Table 需要多個的原因：
- 私有子網路（Private Subnet）的主要目的是提供內部服務的隔離
- 每個可用區域（AZ）的私有子網路需要自己的 NAT 閘道（NAT Gateway）
- 這是因為：
  - NAT 閘道是區域性的服務，每個 AZ 需要自己的 NAT 閘道
  - 為了高可用性，每個 AZ 的私有子網路應該使用該 AZ 的 NAT 閘道
  - 如果跨 AZ 使用 NAT 閘道，可能會造成：
    - 單點故障風險
    - 跨 AZ 的網路延遲
    - 不必要的跨 AZ 流量費用

#### 實際運作方式：
```
公開子網路：
Public Subnet (AZ-a) ──┐
Public Subnet (AZ-b) ──┼──> Single Public Route Table ──> Internet Gateway
Public Subnet (AZ-c) ──┘

私有子網路：
Private Subnet (AZ-a) ──> Private Route Table (AZ-a) ──> NAT Gateway (AZ-a)
Private Subnet (AZ-b) ──> Private Route Table (AZ-b) ──> NAT Gateway (AZ-b)
Private Subnet (AZ-c) ──> Private Route Table (AZ-c) ──> NAT Gateway (AZ-c)
```

#### 這種設計的優點：
- 高可用性：每個 AZ 的私有子網路都有獨立的 NAT 閘道
- 效能優化：避免跨 AZ 的網路流量
- 成本效益：減少跨 AZ 的資料傳輸費用
- 故障隔離：一個 AZ 的問題不會影響其他 AZ 的服務

## 問題二：Public Route Table 就不會有跨 AZ 的單點故障問題嗎？

#### Public Route Table 的單點故障風險：
- 如果公開路由表出現問題，所有公開子網路都會受到影響
- 但這個風險相對較低，因為：
  - 路由表是 AWS 的基礎服務，具有很高的可用性
  - 路由表本身是分散式的，AWS 會自動在區域內複製
  - 路由表的故障機率遠低於 NAT 閘道

#### 為什麼 Public Route Table 可以共用，而 Private Route Table 不行：

##### 公開子網路的流量模式：
```
Public Subnet ──> Internet Gateway ──> Internet
```
- 網際網路閘道（Internet Gateway）是區域性的服務
- 但它是 AWS 的基礎服務，具有極高的可用性
- 流量模式簡單，只需要一個出口點

##### 私有子網路的流量模式：
```
Private Subnet ──> NAT Gateway ──> Internet Gateway ──> Internet
```
- NAT 閘道是區域性的服務，且是付費服務
- 需要處理更多的網路轉換工作
- 流量模式較複雜，需要考慮：
  - 網路位址轉換（NAT）
  - 連線追蹤
  - 流量限制
  - 效能瓶頸

#### 實際的風險比較：
- 公開路由表：
  - 故障機率：極低
  - 影響範圍：所有公開子網路
  - 恢復時間：通常很快
  - 成本：無額外成本

- 私有路由表（如果共用）：
  - 故障機率：較高（因為 NAT 閘道）
  - 影響範圍：所有私有子網路
  - 恢復時間：較長
  - 成本：跨 AZ 流量費用

## 問題三：NAT Gateway 為什麼比較容易故障？

#### NAT Gateway 的複雜性：
- NAT Gateway 需要處理：
  - 網路位址轉換（NAT）
  - 連線追蹤（Connection Tracking）
  - 流量限制（Rate Limiting）
  - 封包轉發（Packet Forwarding）
- 這些操作都需要消耗資源和處理能力

#### 資源限制：
- NAT Gateway 有硬體限制：
  - 每個 NAT Gateway 有最大連線數限制
  - 有頻寬限制
  - 有每秒封包處理限制
- 當這些限制被達到時，可能會導致：
  - 連線被拒絕
  - 封包丟失
  - 效能下降

#### 網路流量模式：
- NAT Gateway 需要處理：
  - 出站流量（Outbound Traffic）
  - 入站流量（Inbound Traffic）
  - 連線狀態維護
- 這些都需要消耗資源和處理能力

#### 故障原因：
- 硬體故障：
  - 網路介面卡故障
  - 處理器過載
  - 記憶體不足
- 軟體問題：
  - 連線表滿載
  - 資源耗盡
  - 軟體錯誤
- 網路問題：
  - 網路擁塞
  - 路由問題
  - DNS 解析問題

#### 與 Internet Gateway 的比較：
- Internet Gateway：
  - 功能簡單：主要是路由轉發
  - 資源需求低：不需要維護連線狀態
  - 硬體要求低：不需要複雜的處理能力
- NAT Gateway：
  - 功能複雜：需要處理 NAT、連線追蹤等
  - 資源需求高：需要維護連線狀態表
  - 硬體要求高：需要較高的處理能力

#### 高可用性設計：
- 每個 AZ 使用獨立的 NAT Gateway：
  - 分散負載
  - 故障隔離
  - 提高可用性
- 使用多個 NAT Gateway：
  - 避免單點故障
  - 提供負載平衡
  - 提高效能

#### 監控和維護：
- 需要監控：
  - 連線數
  - 頻寬使用
  - 封包處理率
- 需要維護：
  - 定期更新
  - 效能優化
  - 故障排除
