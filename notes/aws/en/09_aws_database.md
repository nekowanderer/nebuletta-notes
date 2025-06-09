# AWS Database Services Comparison: RDS vs Aurora

[English](./09_aws_database.md) | [繁體中文](../zh-tw/09_aws_database.md) | [日本語](../ja/09_aws_database.md) | [Back to Index](../README.md)

## Basic Comparison Table

| Feature | RDS (Relational Database Service) | Aurora |
|---------|----------------------------------|---------|
| Service Level | Region Level | Region Level |
| Database Engine | MySQL, PostgreSQL, MariaDB, Oracle, SQL Server | MySQL, PostgreSQL |
| Deployment | Single instance or master-slave architecture | Cluster architecture |
| Scaling Method | Vertical scaling (increasing instance size) | Horizontal scaling (adding read nodes) |
| Backup Method | Automated backups and manual snapshots | Continuous backup and point-in-time recovery |
| Availability | 99.95% | 99.99% |
| Cost | Lower | Higher |
| Maintenance Difficulty | Lower | Higher |

## Real-world Analogies

### RDS Analogy: Traditional Restaurant
- **Head Chef (Primary Database)**: Makes all important decisions and handles main tasks
- **Sous Chef (Standby Database)**: Takes over when the head chef is unavailable
- **Kitchen Equipment (Hardware)**: Fixed size, can be upgraded when needed
- **Menu (Database Engine)**: Can choose different cuisines (MySQL, PostgreSQL, etc.)
- **Operating Hours (Availability)**: Fixed operating hours with occasional maintenance breaks

### Aurora Analogy: Chain Restaurant
- **Central Kitchen (Primary Database)**: Handles core menu and quality control
- **Branch Kitchens (Read Nodes)**: Can quickly scale to handle more orders
- **Automated Equipment (Automation)**: Highly automated kitchen equipment
- **Standardized Processes (Consistency)**: Ensures consistent quality across all branches
- **24/7 Operation (High Availability)**: Almost never closes, always available

## Detailed Comparison

### Architecture Design
#### RDS
- Traditional master-slave architecture
- One primary node and up to five standby nodes
- Standby nodes can be promoted to primary
- Suitable for small to medium applications

#### Aurora
- Distributed architecture
- One primary node and up to 15 read nodes
- Automatic failover
- Suitable for enterprise applications

### Performance
#### RDS
- Standard database performance
- Suitable for general workloads
- Higher latency
- Lower throughput

#### Aurora
- 5x performance of standard MySQL
- Suitable for high-throughput workloads
- Lower latency
- Higher throughput

### Scaling Capability
#### RDS
- Primarily relies on vertical scaling
- Slower read node scaling
- Scaling process may impact performance
- Suitable for stable workloads

#### Aurora
- Supports rapid horizontal scaling
- Read node scaling takes only minutes
- Scaling process doesn't affect performance
- Suitable for dynamic workloads

### Cost Considerations
#### RDS
- Lower initial cost
- Pay-as-you-go pricing
- Suitable for budget-constrained projects
- Lower maintenance costs

#### Aurora
- Higher initial cost
- Cluster-based pricing
- Suitable for high-performance projects
- Higher maintenance costs

### Use Cases
#### RDS is suitable for:
- Small to medium business applications
- Development and testing environments
- Budget-constrained projects
- Standard database requirements

#### Aurora is suitable for:
- Enterprise applications
- Systems requiring high availability
- Applications needing rapid scaling
- Projects with high performance requirements

## Selection Guidelines

### Choose RDS when:
- Budget is limited
- Multiple database engine options are needed
- Workload is relatively stable
- Extreme performance is not required

### Choose Aurora when:
- High availability is required
- Rapid scaling capability is needed
- Workload is highly variable
- High performance is essential

## References
- [Amazon RDS Documentation](https://docs.aws.amazon.com/rds/)
- [Amazon Aurora Documentation](https://docs.aws.amazon.com/aurora/) 
