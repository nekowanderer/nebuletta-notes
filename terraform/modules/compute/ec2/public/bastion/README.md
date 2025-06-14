# Public Bastion EC2 Instance

This module creates a bastion EC2 instance in a public subnet that can be accessed through AWS Systems Manager Session Manager.

```
           +---------------------------------------------------+
           |                       AWS Cloud                   |
           |                                                   |
           |   +---------------------------------------------+ |
           |   |                    VPC                      | |
           |   |                                             | |
           |   |   +---------------------------------------+ | |
           |   |   |           Network ACLs                | | |
           |   |   |                                       | | |
           |   |   |   +--------------+   +--------------+ | | |
           |   |   |   | Public       |   | Private      | | | |
           |   |   |   | Subnet       |   | Subnet       | | | |
           |   |   |   |              |   |              | | | |
           |   |   |   |  +--------+  |   |              | | | |
           |   |   |   |  |  IGW   |  |   |              | | | |
           |   |   |   |  +----+---+  |   |              | | | |
           |   |   |   |       |      |   |              | | | |
           |   |   |   |  +----+---+  |   |              | | | |
           |   |   |   |  |Security|  |   |              | | | |
           |   |   |   |  | Group  |  |   |              | | | |
           |   |   |   |  +----+---+  |   |              | | | |
           |   |   |   |       |      |   |              | | | |
           |   |   |   |  +----+---+  |   |              | | | |
           |   |   |   |  |Bastion |  |   |              | | | |
           |   |   |   |  | EC2    |  |   |              | | | |
           |   |   |   |  +--------+  |   |              | | | |
           |   |   |   |              |   |              | | | |
           |   |   |   +--------------+   +--------------+ | | |
           |   |   |                                       | | |
           |   |   +---------------------------------------+ | |
           |   +---------------------------------------------+ |
           +---------------------------------------------------+
                                    |
                                    |
                    +---------------+---------------+
                    |                               |
                    |           Internet            |
                    |                               |
                    +-------------------------------+
```

## Features

- Creates EC2 instance in public subnet
- Configures security groups to allow outbound traffic
- Creates IAM role and instance profile
- Uses Internet Gateway for external network access
- **Supports AWS Systems Manager Session Manager remote connection** without SSH or public IP

## Session Manager Access

This module configures the EC2 instance to allow secure connection through AWS Systems Manager Session Manager:

1. Attaches AmazonSSMManagedInstanceCore policy to EC2 role
2. No ingress rules required at security group level for Session Manager
3. Can connect to instance directly from AWS Console using Session Manager

### Network Rules

#### Security Group Rules
- **Bastion Security Group**:
  - Ingress Rules:
    - Allow HTTP (80) from any source
    - Allow HTTPS (443) from any source
  - Egress Rules:
    - Allow all outbound traffic (0.0.0.0/0)

#### Service Connection Details
- **Session Manager Connection**: Through AWS Systems Manager, no additional ingress rules needed
- **Internet Access**: Through Internet Gateway
- **VPC Internal Access**: Can access resources in private subnets through VPC internal network

### Prerequisites

To ensure Session Manager works properly, the following IAM permissions must be configured:

- `AmazonSSMManagedInstanceCore` policy attached to EC2 role

## Usage

After obtaining necessary network resources from the state-storage module, use this module as follows:

```hcl
module "bastion" {
  source = "../../../../../../modules/compute/ec2/public/bastion"

  env          = "dev"
  aws_region   = "ap-northeast-1"
  project      = "nebuletta"
  module_name  = "default-vpc-bastion"
  managed_by   = "Terraform"
  
  # Optional parameters
  instance_type = "t3.medium"
  # ami_id       = "ami-12345678" # If not specified, will use latest Amazon Linux 2 AMI
  vpc_id = data.terraform_remote_state.networking.outputs.vpc_id
  public_subnet_ids = data.terraform_remote_state.networking.outputs.public_subnet_ids
}
```

## Input Variables

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| env | Environment Name | string | - | yes |
| aws_region | AWS Region | string | - | yes |
| project | Project Name | string | "nebuletta" | no |
| module_name | Module Name | string | - | yes |
| managed_by | Managed By | string | "Terraform" | no |
| instance_type | EC2 Instance Type | string | "t3.medium" | no |
| ami_id | EC2 Instance AMI ID | string | "" | no |
| vpc_id | VPC ID | string | - | yes |
| public_subnet_ids | List of Public Subnet IDs | list(string) | - | yes |

## Output Variables

| Name | Description |
|------|-------------|
| instance_id | EC2 Instance ID |
| instance_private_ip | EC2 Instance Private IP |
| instance_public_ip | EC2 Instance Public IP |
| security_group_id | Security Group ID |
| iam_role_arn | IAM Role ARN attached to EC2 instance |
| iam_instance_profile_arn | IAM Instance Profile ARN attached to EC2 instance | 
