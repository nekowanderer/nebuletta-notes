# 網路基礎設施設計

[English](../en/README.md) | [繁體中文](README.md) | [日本語](../ja/README.md) | [回到索引](../README.md)

### 網路架構圖
```
+-------------------------------------------------------------------------------------+
|                                                                                     |
|  VPC (10.0.0.0/16)                                                                  |
|                                                                                     |
|  +------------------------------+            +--------------------------------+     |
|  |                              |            |                                |     |
|  |  AZ: ap-northeast-1a         |            |  AZ: ap-northeast-1c           |     |
|  |                              |            |                                |     |
|  |  +------------------------+  |            |  +------------------------+    |     |
|  |  | Public Subnet          |  |            |  | Public Subnet          |    |     |
|  |  | (10.0.1.0/24)          |  |            |  | (10.0.2.0/24)          |    |     |
|  |  |                        |  |            |  |                        |    |     |
|  |  |  +------------------+  |  |            |  |  +------------------+  |    |     |
|  |  |  | NAT Gateway      |  |  |            |  |  | NAT Gateway      |  |    |     |
|  |  |  | + EIP            |  |  |            |  |  | + EIP            |  |    |     |
|  |  |  +--------+---------+  |  |            |  |  +--------+---------+  |    |     |
|  |  |           |            |  |            |  |           |            |    |     |
|  |  +-----------|------------+  |            |  +-----------|------------+    |     |
|  |              |               |            |              |                 |     |
|  |  +-----------v------------+  |            |  +-----------v------------+    |     |
|  |  | Private Subnet         |  |            |  | Private Subnet         |    |     |
|  |  | (10.0.11.0/24)         |  |            |  | (10.0.12.0/24)         |    |     |
|  |  |                        |  |            |  |                        |    |     |
|  |  | +--------------------+ |  |            |  | +--------------------+ |    |     |
|  |  | | VPC Endpoint ENIs  | |  |            |  | | VPC Endpoint ENIs  | |    |     |
|  |  | | (Interface 類型)    | |  |            |  | | (Interface 類型)    | |    |     |
|  |  | +--------------------+ |  |            |  | +--------------------+ |    |     |
|  |  +------------------------+  |            |  +------------------------+    |     |
|  |                              |            |                                |     |
|  +------------------------------+            +--------------------------------+     |
|                                                                                     |
|  +-------------------------+         +-------------------------+        +---------+ |
|  | Route Table (public)    |         | Route Tables (private)  | <----> | Gateway | |
|  | 0.0.0.0/0 -> IGW        |         | 0.0.0.0/0 -> NAT        |        | VPC     | |
|  +-----------|------------+         +--------------------------+        | Endpoints| |
|              |                                                          | (S3,     | |
|              v                                                          | DynamoDB)| |
|  +-------------------------+                                            +---------+ |
|  | Internet Gateway        |                                                        |
|  +-----------|------------+                                                         |
|              |                                                                      |
+--------------|----------------------------------------------------------------------+
               |
               v
         +-------------+
         |  Internet   |
         +-------------+
```

> **圖表限制說明：** 上面的圖表並未明確顯示可用區域（AZ）與 Internet Gateway（IGW）或路由表之間的直接連線，原因如下：
> 
> 1. **概念層次差異：** 在 AWS 架構中，Internet Gateway 和路由表在邏輯上是附加到整個 VPC，而不是直接附加到 AZ。AZ 是 AWS 的實體基礎設施區隔，而 IGW 和路由表則是邏輯配置。
> 
> 2. **實際關係是：**
>    - Internet Gateway 附加在 VPC 層級
>    - 路由表配置在 VPC 層級，然後與特定子網路關聯
>    - 子網路存在於特定的 AZ 中
> 
> 3. **圖表清晰度考量：** 加入這些交錯的連線會使圖表變得更加複雜且可能造成混淆。目前的表示方式著重於流量的功能流向，而非每個邏輯關係。
> 
> 若需要更全面地視覺化這些關係，建議使用專業繪圖工具，如 AWS 架構圖工具、Lucidchart 或 draw.io。

### 架構樹狀圖
```
aws_vpc.this
├── aws_internet_gateway.this  ───→ 公共出入口
├── aws_subnet.public[N]
│   └── 綁定 aws_route_table.public → 指定走 IGW
│       └── aws_route.igw
├── aws_subnet.private[N]
│   └── 綁定 aws_route_table.private[N] → 指定走 NAT
│       └── aws_route.nat[N]
├── aws_nat_gateway.this[N]
│   ├── 放在 public subnet
│   └── 用 aws_eip.nat[N] 作為出口位址
├── aws_vpc_endpoint (Gateway 類型)
│   ├── S3 (連接所有路由表)
│   └── DynamoDB (連接所有路由表)
├── aws_vpc_endpoint (Interface 類型)
│   ├── ECR API (連接 private subnet)
│   ├── ECR Docker (連接 private subnet)
│   └── CloudWatch Logs (連接 private subnet)
├── aws_network_acl.public
│   └── 允許所有出入流量
├── aws_network_acl.private
│   ├── 入口只允許 VPC 內部流量
│   └── 出口允許所有流量
└── aws_flow_log.vpc_flow_log
    └── 將所有網路流量記錄到 CloudWatch Logs
```

### 參考順序與邏輯
1. aws_vpc.this
 - 每個資源都會依附在某個 VPC 上。這裡的 VPC 是基礎，也是所有其他網路元件的「母體」。

2. aws_internet_gateway.this
 - 跟 VPC 一對一綁定，讓這個 VPC 的 public subnet 有對外的網路出入口。
 - IGW 是一種 AWS 提供的「對外 NAT」設備，給 Public Subnet 用。

3. aws_subnet.public + aws_subnet.private
 - Public Subnet 設定了 map_public_ip_on_launch = true，代表這裡啟動的 EC2 會自動配一個 public IP。
 - 每個 Subnet 指定了一個 AZ，也指定了 CIDR（區域位址範圍）。

4. aws_eip.nat
 - EIP（Elastic IP）= 固定不變的公共 IP 位址
 - AWS 預設給 NAT Gateway 的 IP 是動態的，但如果你想確保 private subnet 每次出去的 IP 是固定的（例如連線到外部資料庫白名單），就要搭配 EIP。
 - 這裡 count = length(var.azs)，意思是為每個 AZ 都準備一個 NAT 用的固定 IP。

5. aws_nat_gateway.this
 - 這是 private subnet 上網的「代理人」，擋在中間。
 - 每個 NAT Gateway：
  - 要用一個 public subnet 當作它的起點（subnet_id = aws_subnet.public[count.index].id）
  - 要綁定一個固定 IP，也就是上面的 EIP。
 - 這是私有子網路能「安全上網」的關鍵。

6. aws_route_table.public + aws_route.igw
 - 建一個「公共區域」的路由表，指定「如果目標是 0.0.0.0/0（全世界），就經由 IGW 出去」。
 - 然後用 aws_route_table_association.public 把這個路由表指定給每個 public subnet。

7. aws_route_table.private + aws_route.nat
 - 為每個 private subnet 各建一張獨立的路由表。
 - 指定「如果要連到外部世界，請透過 NAT Gateway 出去」。
 - 路由表也需要透過 route_table_association 綁定到子網路上。

8. aws_vpc_endpoint (Gateway 類型)
 - 提供從 VPC 內部直接連接到 AWS 服務的方式，不需要經過公網。
 - S3 和 DynamoDB 端點是 Gateway 類型，直接添加到路由表中。
 - 這種連線方式可以節省 NAT Gateway 費用，增加安全性和效能。

9. aws_vpc_endpoint (Interface 類型)
 - 為 ECR API、ECR Docker 和 CloudWatch Logs 建立的端點。
 - 這些是 Interface 類型，在每個指定的 private subnet 放置一個 ENI（彈性網路介面）。
 - 需要指定安全群組以控制流量。
 - 啟用了 private DNS，這意味著標準 AWS 服務 DNS 名稱會自動解析到 VPC 端點 IP。

10. aws_network_acl (Network Access Control List)
 - 作為子網路層級的防火牆，與安全群組（Security Group）相輔相成。
 - 公共子網路的 NACL 允許所有流量進出，適合需要與外界直接通訊的服務。
 - 私有子網路的 NACL 僅允許 VPC 內部流量進入，但允許所有流量出去，增強安全性。

11. aws_flow_log.vpc_flow_log
 - 記錄所有 VPC 網路流量，對於安全審計和故障排除非常有用。
 - 日誌存儲在 CloudWatch Logs 中，保留期為 5 天。
 - 需要特定的 IAM 角色來授權 VPC 將日誌寫入 CloudWatch。

### CIDR 配置方案
- VPC CIDR: 10.0.0.0/16（提供 65,536 個 IP 位址）
- 公共子網路:
  - ap-northeast-1a: 10.0.1.0/24（256 個 IP）
  - ap-northeast-1c: 10.0.2.0/24（256 個 IP）
- 私有子網路:
  - ap-northeast-1a: 10.0.11.0/24（256 個 IP）
  - ap-northeast-1c: 10.0.12.0/24（256 個 IP）

### 總結
1. VPC 是地 → Subnet 是分區
2. IGW 拉出上網線 → 給 Public 區使用
3. NAT Gateway + EIP 組成 Private 區的「防火牆 + 上網管道」
4. Route Table 只是告訴每個子網路「怎麼出門」
5. VPC Endpoints 讓私有子網路可以安全地訪問 AWS 服務，不需經過公網
6. Network ACLs 提供子網路層級的流量控制
7. VPC Flow Logs 記錄流量以便監控和故障排除
