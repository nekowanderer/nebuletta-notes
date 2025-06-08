# AWS Networking Basics

[English](./09_aws_networking_basics.md) | [繁體中文](../zh-tw/09_aws_networking_basics.md) | [日本語](../ja/09_aws_networking_basics.md) | [Back to Index](../README.md)

## Basic Service Analogies
- **ELB (Elastic Load Balancer)**: Like a greeter at the entrance, directing customers to available staff
  - Operates at the region level
- **EC2**: The actual service instances, like staff/back-end baristas
- **SQS/SNS**: Buffers between services, similar to a display board showing pending orders between cashiers and baristas

## High Availability (HA)
- Concept: Like opening multiple branches of a coffee shop, allowing customers to visit other locations if one is unavailable
- Key factors when choosing a Region:
  - Compliance
  - Proximity
  - Feature Availability
  - Pricing

## VPC (Virtual Private Cloud)
- Definition: Resource isolation concept in AWS
- Key Components:
  - **Internet Gateway (IGW)**
    - Function: External access point for VPC
    - Analogy: The main entrance of a coffee shop
  - **Virtual Private Gateway (VGW)**
    - Function: Provides secure VPN connectivity (acts as the Amazon-side VPN endpoint in Site-to-Site VPN connections, can be attached to a single VPC)
    - Use Case: Enterprise internal networks requiring secure connection to VPC
  - **AWS Direct Connect**
    - Function: Provides physical dedicated line connection between enterprise data centers and AWS VPC

## Subnet Network Segmentation
#### Types
- **Public Subnet**
  - Characteristics: Direct access to IGW
  - Analogy: Front counter area of a coffee shop
- **Private Subnet**
  - Characteristics: No direct access to IGW
  - Analogy: Back-end work area of a coffee shop

#### Network Security Mechanism Comparison

| Feature | Security Group (SG) | Network ACL (ACL) |
|---------|-------------------|------------------|
| Operation Level | Instance level | Subnet level |
| State | Stateful | Stateless |
| Default Rules | Deny all inbound traffic | Allow all traffic |
| Traffic Direction | Inbound only | Both inbound and outbound |
| Analogy | Building security guard | Immigration officer |
| Rule Limit | Fewer | More |
| Rule Priority | All rules evaluated | Sequential evaluation |

## Network Traffic Flow Example
Process when a packet travels from subnet1.instanceA to subnet2.instanceB:

1. Sending Phase:
   - subnet1.instanceA's SG allows outbound traffic
   - subnet1's ACL checks and permits
   - subnet2's ACL checks and permits
   - subnet2.instanceB's SG checks and permits

2. Response Phase:
   - subnet2.instanceB's SG allows outbound traffic
   - subnet2's ACL checks and permits
   - subnet1's ACL checks and permits
   - subnet1.instanceA's SG automatically permits (stateful)

## Terraform Resource Relationships
Resource relationships:
- EC2 instance needs to define associated subnet/SG
- subnet needs to define associated VPC
- SG needs to define associated VPC
- ACL needs to define associated VPC + subnet

Relationship diagram:
```
VPC
├── Subnet
│   ├── ACL
│   └── EC2 Instance
│       └── Security Group
└── Security Group
```

## References
- [Control subnet traffic with network access control lists](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html)
- [Control traffic to your AWS resources using security groups](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html) 