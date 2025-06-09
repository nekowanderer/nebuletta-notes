# AWS Storage Services Comparison

[English](./08_aws_ebs_efs_s3.md) | [繁體中文](../zh-tw/08_aws_ebs_efs_s3.md) | [日本語](../ja/08_aws_ebs_efs_s3.md) | [Back to Index](../README.md)

## Basic Comparison Table

| Feature | EBS (Elastic Block Store) | EFS (Elastic File System) | S3 (Simple Storage Service) |
|---------|-------------------------|-------------------------|---------------------------|
| Service Level | AZ Level | Region Level | Region Level |
| Use Case | Block storage for direct disk access | File system for shared access | Object storage for static files, backups |
| Access Method | Block-level access | File system access | REST API access |
| Connection | Only to EC2 in same AZ | Cross-AZ connection | API or HTTP/HTTPS access |
| Data Consistency | Strong consistency | Strong consistency | Eventual consistency |
| Availability | 99.999% | 99.999% | 99.999999999% |
| Durability | 99.999% | 99.999999999% | 99.999999999% |
| Scalability | Up to 16TB per volume | Auto-scaling, no capacity limit | No capacity limit |
| Backup Method | Snapshots | Backups | Versioning |
| Cost Model | Based on capacity and IOPS | Based on usage | Based on storage and requests |

## Detailed Description

### EBS (Elastic Block Store)
- **Primary Uses**:
  - EC2 instance system disks
  - Database storage
  - Low-latency applications
- **Features**:
  - Block-level storage
  - Snapshot capability
  - Encryption support
  - Resizable volumes
- **Limitations**:
  - Only connects to EC2 in same AZ
  - Capacity limits
  - Manual scaling required

### EFS (Elastic File System)
- **Primary Uses**:
  - Shared file systems
  - Content management systems
  - Multi-EC2 access applications
- **Features**:
  - Auto-scaling
  - NFSv4 protocol support
  - Cross-AZ availability
  - Lifecycle management
- **Limitations**:
  - Higher latency than EBS
  - Higher cost than EBS

### S3 (Simple Storage Service)
- **Primary Uses**:
  - Static website hosting
  - Backup and archiving
  - Big data analytics
  - Content distribution
- **Features**:
  - Unlimited scaling
  - High durability
  - Multiple storage classes
  - Rich access controls
- **Limitations**:
  - Not suitable for low-latency applications
  - Not suitable for file system applications

## Usage Recommendations

### When to Choose EBS
- Applications requiring block-level access
- Low-latency databases
- EC2 instances needing system disks

### When to Choose EFS
- Applications requiring shared file systems
- Scenarios needing multi-EC2 access
- File systems requiring auto-scaling

### When to Choose S3
- Storing large amounts of static files
- Backup and archiving needs
- Content distribution
- Unlimited storage space requirements

## References
- [Amazon EBS Documentation](https://docs.aws.amazon.com/ebs/)
- [Amazon EFS Documentation](https://docs.aws.amazon.com/efs/)
- [Amazon S3 Documentation](https://docs.aws.amazon.com/s3/) 
