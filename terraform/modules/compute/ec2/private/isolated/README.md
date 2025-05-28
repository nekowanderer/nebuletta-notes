# Private Isolated EC2 Instance

This module creates an EC2 instance in a private subnet that can connect to external networks and use VPC endpoints.

```
           +----------------------------------------------------------+
           |                       AWS Cloud                          |
           |                                                          |
           |   +----------------------------------------------------+ |
           |   |                    VPC                             | |
           |   |                                                    | |
           |   |   +----------------------------------------------+ | |
           |   |   |           Network ACLs                       | | |
           |   |   |                                              | | |
           |   |   |   +--------------+   +---------------------+ | | |
           |   |   |   | Public       |   | Private             | | | |
           |   |   |   | Subnet       |   | Subnet              | | | |
           |   |   |   |              |   |                     | | | |
           |   |   |   |              |   |  +----------------+ | | | |
           |   |   |   |              |   |  | Security Group | | | | |
           |   |   |   |              |   |  |   +----+       | | | | |
           |   |   |   |              |   |  |   |EC2 |       | | | | |
           |   |   |   |              |   |  |   +----+       | | | | |
           |   |   |   |              |   |  +----------------+ | | | |
           |   |   |   +--------------+   +---------------------+ | | |
           |   |   |                                              | | |
           |   |   +----------------------------------------------+ | |
           |   +----------------------------------------------------+ |
           +----------------------------------------------------------+
                                    |
                                    |
                    +---------------+---------------+
                    |                               |
                    |           Internet            |
                    |                               |
                    +-------------------------------+
```

## Features

- Creates EC2 instance in private subnet
- Configures security groups to allow outbound traffic
- Creates IAM role and instance profile
- Uses existing VPC endpoints for AWS service access
- **Supports AWS Systems Manager Session Manager remote connection** without SSH or public IP

## Session Manager Access

This module configures the EC2 instance to allow secure connection through AWS Systems Manager Session Manager:

1. Attaches AmazonSSMManagedInstanceCore policy to EC2 role
2. No ingress rules required at security group level for Session Manager (as it uses VPC Endpoints)
3. Can connect to instance directly from AWS Console using Session Manager

### Network Rules

#### Network ACLs Rules
- **Private Subnet Network ACLs (see `infra/networking/acls_private.tf` for details)**:
  - Ingress Rules:
    - Allow ICMP from any source (ping, for network diagnostics)
    - Allow HTTPS (TCP 443) from any source
    - Allow ephemeral ports (1024-65535) from any source (for response traffic)
    - Allow all traffic from VPC CIDR (internal service communication)
  - Egress Rules:
    - Allow all ICMP outbound traffic (network diagnostics)
    - Allow HTTP (80) and HTTPS (443) outbound traffic (web browsing, package management)
    - Allow DNS queries (TCP/UDP 53)
    - Allow SSH outbound connections (TCP 22)
    - Allow all outbound traffic to VPC CIDR (internal service communication)
    - Allow ephemeral port outbound traffic (1024-65535, for client connections)

#### Service Connection Details
- **Session Manager Connection**: Through VPC Endpoints, no additional ingress rules needed
- **Other Service Connections**:
  - Allow ICMP (ping) for network diagnostics
  - Allow HTTP/HTTPS for git clone, wget, etc.
  - Allow DNS queries
  - Other necessary outbound traffic

### Prerequisites

To ensure Session Manager works properly in private subnet, the following VPC endpoints must be configured:

- com.amazonaws.[region].ssm
- com.amazonaws.[region].ec2messages
- com.amazonaws.[region].ssmmessages

If using the terraform/infra/networking module, these endpoints may need to be added to that module.

## Usage

After obtaining necessary network resources from the infra/networking module, use this module as follows:

```hcl
module "ec2_instance" {
  source = "../infra/ec2_instance"

  module_name    = "nebuletta"
  environment     = "dev"
  vpc_id          = module.networking.vpc_id
  private_subnet_ids = module.networking.private_subnet_ids
  
  # Optional parameters
  instance_type   = "t3.micro"
  key_name        = "my-ssh-key"  # If you want to keep SSH access option
  # ami_id         = "ami-12345678" # If not specified, will use latest Amazon Linux 2 AMI
}
```

## Input Variables

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| aws_region | AWS Region | string | "ap-northeast-1" | no |
| module_name | Project Name | string | - | yes |
| environment | Environment Name | string | - | yes |
| vpc_id | VPC ID | string | - | yes |
| private_subnet_ids | List of Private Subnet IDs | list(string) | - | yes |
| instance_type | EC2 Instance Type | string | "t3.micro" | no |
| ami_id | EC2 Instance AMI ID | string | "" | no |
| key_name | SSH Key Name | string | "" | no |
| additional_security_group_ids | Additional Security Group IDs to attach to instance | list(string) | [] | no |

## Output Variables

| Name | Description |
|------|-------------|
| ec2_instance_id | EC2 Instance ID |
| ec2_instance_private_ip | EC2 Instance Private IP |
| ec2_security_group_id | EC2 Security Group ID |
| ec2_iam_role_name | IAM Role Name attached to EC2 instance |
| ec2_iam_role_arn | IAM Role ARN attached to EC2 instance |
