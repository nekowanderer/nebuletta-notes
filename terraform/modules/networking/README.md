# Infra Network Design

This directory contains network infrastructure setup for the project.

## Documentation

Documentation is available in the following languages:

- [English](docs/en/README.md)
- [繁體中文](docs/zh-tw/README.md)
- [日本語](docs/ja/README.md)

### Network Architecture Diagram
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
|  |  | | (Interface Type)   | |  |            |  | | (Interface Type)   | |    |     |
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

> **Note on Diagram Limitations:** The diagram above does not explicitly show direct connections between Availability Zones (AZs) and the Internet Gateway (IGW) or Route Tables for several reasons:
> 
> 1. **Conceptual Layering:** In AWS architecture, Internet Gateways and Route Tables are logically attached to the VPC as a whole, not directly to AZs. AZs are AWS's physical infrastructure segmentation, while network components like IGW and Route Tables are logical configurations.
> 
> 2. **Actual Relationship:** The correct relationship is:
>    - Internet Gateway is attached at the VPC level
>    - Route Tables are configured at the VPC level, then associated with specific subnets
>    - Subnets exist within specific AZs
> 
> 3. **Diagram Clarity:** Adding these cross-connections would make the diagram significantly more complex and potentially confusing. The current representation focuses on the functional flow of traffic rather than every logical relationship.
> 
> For a more comprehensive visualization of these relationships, specialized diagramming tools like AWS Architecture Diagrams, Lucidchart, or draw.io would be recommended.

### Architecture Tree Structure
```
aws_vpc.this
├── aws_internet_gateway.this  ───→ Public entry/exit point
├── aws_subnet.public[N]
│   └── Bound to aws_route_table.public → Specified to use IGW
│       └── aws_route.igw
├── aws_subnet.private[N]
│   └── Bound to aws_route_table.private[N] → Specified to use NAT
│       └── aws_route.nat[N]
├── aws_nat_gateway.this[N]
│   ├── Placed in public subnet
│   └── Using aws_eip.nat[N] as exit address
├── aws_vpc_endpoint (Gateway Type)
│   ├── S3 (Connected to all route tables)
│   └── DynamoDB (Connected to all route tables)
├── aws_vpc_endpoint (Interface Type)
│   ├── ECR API (Connected to private subnet)
│   ├── ECR Docker (Connected to private subnet)
│   └── CloudWatch Logs (Connected to private subnet)
├── aws_network_acl.public
│   └── Allows all inbound and outbound traffic
├── aws_network_acl.private
│   ├── Inbound only allows VPC internal traffic
│   └── Outbound allows all traffic
└── aws_flow_log.vpc_flow_log
    └── Records all network traffic to CloudWatch Logs
```

### Reference Sequence and Logic
1. aws_vpc.this
 - Each resource is attached to a VPC. The VPC is the foundation and the "parent" of all other network components.

2. aws_internet_gateway.this
 - Bound one-to-one with the VPC, providing the public subnet with external network access.
 - IGW is an "external NAT" device provided by AWS for Public Subnets.

3. aws_subnet.public + aws_subnet.private
 - Public Subnet is configured with map_public_ip_on_launch = true, meaning EC2 instances launched here will automatically be assigned a public IP.
 - Each Subnet specifies an AZ and a CIDR block (regional address range).

4. aws_eip.nat
 - EIP (Elastic IP) = Fixed public IP address
 - AWS assigns dynamic IPs to NAT Gateways by default, but if you want to ensure that private subnets always use the same outgoing IP (e.g., for whitelisting with external databases), you need an EIP.
 - Here count = length(var.azs), meaning we prepare one fixed IP for NAT in each AZ.

5. aws_nat_gateway.this
 - This is the "proxy" for private subnets to access the internet, acting as an intermediary.
 - Each NAT Gateway:
  - Uses a public subnet as its starting point (subnet_id = aws_subnet.public[count.index].id)
  - Is bound to a fixed IP, which is the EIP mentioned above.
 - This is crucial for private subnets to "safely access the internet".

6. aws_route_table.public + aws_route.igw
 - Creates a "public area" route table, specifying "if the destination is 0.0.0.0/0 (the entire world), go through IGW".
 - Then uses aws_route_table_association.public to assign this route table to each public subnet.

7. aws_route_table.private + aws_route.nat
 - Creates separate route tables for each private subnet.
 - Specifies "if connecting to the external world, go through NAT Gateway".
 - Route tables need to be bound to subnets through route_table_association.

8. aws_vpc_endpoint (Gateway Type)
 - Provides a way to directly connect from within the VPC to AWS services without going through the public internet.
 - S3 and DynamoDB endpoints are Gateway type, added directly to route tables.
 - This connection method can save NAT Gateway costs and increase security and performance.

9. aws_vpc_endpoint (Interface Type)
 - Endpoints created for ECR API, ECR Docker, and CloudWatch Logs.
 - These are Interface type, placing an ENI (Elastic Network Interface) in each specified private subnet.
 - Requires specifying a security group to control traffic.
 - Private DNS is enabled, meaning standard AWS service DNS names will automatically resolve to VPC endpoint IPs.

10. aws_network_acl (Network Access Control List)
 - Acts as a subnet-level firewall, complementing Security Groups.
 - The NACL for public subnets allows all traffic in and out, suitable for services that need direct communication with the outside world.
 - The NACL for private subnets only allows inbound traffic from within the VPC but allows all outbound traffic, enhancing security.

11. aws_flow_log.vpc_flow_log
 - Records all VPC network traffic, useful for security audits and troubleshooting.
 - Logs are stored in CloudWatch Logs with a 5-day retention period.
 - Requires a specific IAM role to authorize the VPC to write logs to CloudWatch.

### CIDR Allocation Plan
- VPC CIDR: 10.0.0.0/16 (Provides 65,536 IP addresses)
- Public Subnets:
  - ap-northeast-1a: 10.0.1.0/24 (256 IPs)
  - ap-northeast-1c: 10.0.2.0/24 (256 IPs)
- Private Subnets:
  - ap-northeast-1a: 10.0.11.0/24 (256 IPs)
  - ap-northeast-1c: 10.0.12.0/24 (256 IPs)

### Summary
1. VPC is the land → Subnet is the zone
2. IGW establishes internet connection → For Public area use
3. NAT Gateway + EIP form the "firewall + internet access channel" for Private area
4. Route Table simply tells each subnet "how to go out"
5. VPC Endpoints allow private subnets to securely access AWS services without going through the public internet
6. Network ACLs provide subnet-level traffic control
7. VPC Flow Logs record traffic for monitoring and troubleshooting 
