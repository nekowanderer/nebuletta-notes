# AWS EKS and EFS Persistent Storage Integration

[English](../en/16_aws_eks_and_efs.md) | [繁體中文](../zh-tw/16_aws_eks_and_efs.md) | [日本語](../ja/16_aws_eks_and_efs.md) | [Back to Index](../README.md)

## Overview

In a Kubernetes environment, the lifecycle of Pods is ephemeral, and data within containers is lost when Pods are restarted or migrated. To achieve persistent data storage, AWS EKS provides an integration solution with EFS (Elastic File System) through CSI (Container Storage Interface) drivers to manage storage resources.

## Architecture Diagram

```                                                          
              Admin EC2                                   Cluster                                                                     
  +-------------------------------+    +------------------------------------------------+                                             
  | +--------+                    |    |                                                |                                             
  | | eksctl |                    |    |                                 PV             |                                             
  | +--------+                    |    |                             +--------+     +-------+      +----------+      +-------------+  
  |  |                            |    |         Pod                 | static |---->|  csi  |----->| Security |----->| AWS EFS     |  
  |  |--create k8s cluster -------|--->|  +---------------+          +--------+     +-------+      | Group    |      | File System |  
  |  |                            |    |  | +-----------+ |               |             |          +----------+      +-------------+  
  |  +--create cluster resources -|--->|  | | container | |               |             |                                             
  |                               |    |  | +-----------+ |       +--------------+      |                                             
  | +---------+                   |    |  |   |           |       | StorageClass |      |                                             
  | | kubectl |                   |    |  |   |-image     |       +--------------+      |                                             
  | +---------+                   |    |  |   |           |               |             |                                             
  |  |                            |    |  |   +-Volume ---|-----\         |             |                                             
  |  +--access k8s cluster -------|--->|  +---------------+      ---\  +-----+          |                                             
  |                               |    |                             ->| PVC |          |                                             
  +-------------------------------+    |                               +-----+          |                                             
                                       |                                                |                                             
                                       |                                                |                                             
                                       +------------------------------------------------+                                                                
```

## Core Component Explanation

#### CSI (Container Storage Interface) Driver

CSI is a standardized interface between Kubernetes and storage systems. The AWS EFS CSI Driver is specifically designed for EKS and is responsible for:

- **Storage Resource Management**: Dynamically create and manage EFS file systems
- **Mount Operations**: Mount EFS file systems into Pods
- **Access Control**: Manage EFS access permissions and security groups
- **Lifecycle Management**: Handle creation, deletion, and updates of storage resources

#### StorageClass

StorageClass defines a template for dynamic storage provisioning:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-xxxxx
  directoryPerms: "700"
```

**Important Parameter Explanation:**
- `provisioner`: Specifies the use of EFS CSI driver
- `provisioningMode`: Provisioning mode (efs-ap means using Access Point)
- `fileSystemId`: EFS file system ID
- `directoryPerms`: Directory permission settings

#### PersistentVolumeClaim (PVC)

PVC is a declaration for Pods to request storage resources:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```

**Key Features:**
- `ReadWriteMany`: Supports multiple Pods reading and writing simultaneously
- `storageClassName`: Specifies the StorageClass to use
- `storage`: Requested storage capacity (EFS is actually charged based on usage)

#### PersistentVolume (PV)

PV is the actual storage resource, dynamically created by the CSI driver:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-xxxxx::fsap-xxxxx
```

## Advantages of CSI Driver

#### Standardized Interface
- Follows Kubernetes CSI standards
- Compatible with other storage systems
- Simplifies storage management processes

#### Dynamic Provisioning
- Creates storage resources on demand
- Automatically manages PV lifecycle
- Reduces manual configuration work

#### Multi-tenancy Support
- Achieves isolation through Access Points
- Supports different permission settings
- Improves security

#### High Availability
- Cross-Availability Zone data replication
- Automatic failover
- 99.99% availability guarantee

## Security Considerations

#### Network Security
- Use VPC endpoints to reduce network latency
- Configure appropriate security group rules
- Restrict EFS network access

#### Access Control
- Use IAM roles for authentication
- Configure EFS Access Point permissions
- Implement least privilege principle

#### Data Encryption
- Enable encryption in transit (TLS)
- Enable encryption at rest (KMS)
- Regularly rotate encryption keys

## Best Practices

#### Performance Optimization
- Use EFS performance modes (General Purpose or Max I/O)
- Configure appropriate throughput modes
- Monitor I/O performance metrics

#### Cost Management
- Choose appropriate storage classes
- Monitor storage usage
- Implement lifecycle policies

#### Backup Strategy
- Regularly create EFS snapshots
- Test disaster recovery procedures
- Establish data retention policies

## Troubleshooting

#### Common Issues

1. **PVC Cannot Bind**
- Check StorageClass configuration
- Verify EFS file system status
- Confirm CSI driver is running properly

2. **Pod Cannot Mount EFS**
- Check security group rules
- Verify network connectivity
- Confirm IAM permission settings

3. **Performance Issues**
- Monitor EFS performance metrics
- Adjust application I/O patterns
- Consider using EFS performance modes

## Summary

Through the AWS EFS CSI Driver, EKS clusters can easily achieve persistent data storage. The CSI driver provides a standardized interface that simplifies storage management processes while ensuring data reliability and security. Proper configuration and use of EFS can provide high-performance, high-availability storage solutions for containerized applications.