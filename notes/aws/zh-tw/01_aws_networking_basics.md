# AWS 網路基礎概念

[English](../en/01_aws_networking_basics.md) | [繁體中文](./01_aws_networking_basics.md) | [日本語](../ja/01_aws_networking_basics.md) | [回到索引](../README.md)

## 基本服務類比
- **ELB (Elastic Load Balancer)**: 門口的迎賓員，負責將流量分配到有容量的伺服器
  - 運作在 region 層級
- **EC2**: 實際執行服務的實例，如同店員/後台的咖啡師
- **SQS/SNS**: 服務之間的緩衝區，類似收銀員與咖啡師之間的待處理訂單顯示板

## 高可用性 (High Availability)
- 概念：如同咖啡店開設分店，當一間店無法服務時，客戶可以轉往其他分店
- 選擇 Region 的考量因素：
  - 法規遵循 (Compliance)
  - 地理位置接近度 (Proximity)
  - 功能可用性 (Feature Availability)
  - 價格 (Pricing)

## VPC (Virtual Private Cloud)
- 定義：AWS 中的資源隔離概念
- 主要元件：
  - **Internet Gateway (IGW)**
    - 功能：VPC 對外開放的出入口
    - 類比：咖啡店的大門
  - **Virtual Private Gateway（VGW）**
    - 功能：提供安全的 VPN 連接（是 Site-to-Site VPN 連接中 Amazon 端的 VPN 終端，可以附加到單個 VPC。）
    - 適用場景：企業內部網路需要安全連接至 VPC
  - **AWS Direct Connect**
    - 功能：提供企業資料中心與 AWS VPC 之間的物理專線連接

## Subnet 網路分割
#### 類型
- **Public Subnet**
  - 特點：可直接存取 IGW
  - 類比：咖啡店的櫃檯區域
- **Private Subnet**
  - 特點：無法直接存取 IGW
  - 類比：咖啡店的後台工作區

#### 網路安全機制比較

| 特性 | Security Group (SG) | Network ACL (ACL) |
|------|-------------------|------------------|
| 運作層級 | Instance 層級 | Subnet 層級 |
| 狀態 | 有狀態 (Stateful) | 無狀態 (Stateless) |
| 預設規則 | 拒絕所有入站流量 | 允許所有流量 |
| 檢查方向 | 只檢查入站流量 | 檢查入站和出站流量 |
| 類比 | 大樓警衛 | 海關移民官 |
| 規則數量限制 | 較少 | 較多 |
| 規則優先順序 | 全部評估 | 按順序評估 |

## 網路流量流程範例
當封包從 subnet1.instanceA 傳送到 subnet2.instanceB 的過程：

1. 發送階段：
   - subnet1.instanceA 的 SG 允許出站流量
   - subnet1 的 ACL 檢查並放行
   - subnet2 的 ACL 檢查並放行
   - subnet2.instanceB 的 SG 檢查並放行

2. 回應階段：
   - subnet2.instanceB 的 SG 允許出站流量
   - subnet2 的 ACL 檢查並放行
   - subnet1 的 ACL 檢查並放行
   - subnet1.instanceA 的 SG 自動放行（有狀態）

## Terraform 資源關聯
資源之間的關聯關係：
- EC2 instance 需要定義相關的 subnet/SG
- subnet 需要定義相關的 VPC
- SG 需要定義相關的 VPC
- ACL 需要定義相關的 VPC + subnet

關聯圖示意：
```
VPC
├── Subnet
│   ├── ACL
│   └── EC2 Instance
│       └── Security Group
└── Security Group
```

## 參考文獻
- [Control subnet traffic with network access control lists](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html)
- [Control traffic to your AWS resources using security groups](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)
