# EKS Fargate Cluster Architecture and Mechanism

[English](../en/15_aws_eks_fargate_cluster.md) | [繁體中文](../zh-tw/15_aws_eks_fargate_cluster.md) | [日本語](../ja/15_aws_eks_fargate_cluster.md) | [Back to Index](../README.md)

### What is EKS Fargate

EKS Fargate is a serverless container runtime provided by Amazon that allows users to run Kubernetes Pods without the need to manage EC2 instances. Fargate automatically allocates appropriate computing resources for each Pod and takes complete control of the underlying infrastructure management.

### Architecture Diagram
```                                                                                                                                                                                                                                                                        
                                                           AWS EKS                                                                       

              Admin EC2                                    Cluster                                     Fargate Computing                 
   +-------------------------------+    +------------------------------------------+      +------------------------------------------+   
   | +--------+                    |    |                                          |      | +-------------+  +---------------------+ |   
   | | eksctl |                    |    |                                          |      | | Master Node |  | Master Replica Node | |   
   | +--------+                    |    | +-------------+      +-----------------+ |      | +-------------+  +---------------------+ |   
   |  |                            |    | | Pod: my-pod |      | Pod: my-pod     | |      +------------------------------------------+   
   |  |--create k8s cluster -------|--->| | ns: app-ns  |      | ns: kube-system | |                                                     
   |  |                            |    | +------|------+      +--------|--------+ |                   Fargate Computing                 
   |  +--create cluster resources -|--->|        |                      |          |      +------------------------------------------+   
   |                               |    |        |                      |          |      |  +-------------+     +-------------+     |   
   +-------------------------------+    +--------|----------------------|----------+      |  | Worker Node |     | Worker Node |     |   
                                                 \                      \                 |  +-------------+     +-------------+     |   
                                                  |                      |                |     +-------------+     +-------------+  |   
                                         +-----------------+    +-----------------+       |   ->| Worker Node |     | Worker Node |  |   
                                         | Fargate Profile |    | Fargate Profile |-------|--/  +-------------+     +-------------+  |   
                                         +-----------------+    +-----------------+       |                                          |   
                                                  |                                       |    +-------------+    +-------------+    |   
                                                  ----------------------------------------|--->| Worker Node |    | Worker Node |    |   
                                                                                          |    +-------------+    +-------------+    |   
                                                                                          +------------------------------------------+   
```

### Core Architecture Components

Based on the architecture diagram, EKS Fargate contains the following key components:

- **Resource Provider**: The AWS resource provider on the left side, including Admin EC2 and eksctl tools
- **Kubernetes Cluster**: The K8s cluster in the middle, containing various Pods and Namespaces
- **Fargate Profile**: The Fargate configuration that defines which Pods should run on Fargate
- **Fargate Computing**: The actual Fargate computing resources, divided into Master Nodes and Worker Nodes

### EKS Fargate Operating Mechanism

```
Admin EC2 → eksctl → Kubernetes Cluster → Fargate Profile → Fargate Computing
```

1. **Cluster Creation**: Create the K8s cluster using the eksctl tool on Admin EC2
2. **Resource Configuration**: eksctl creates various resources required for the cluster
3. **Pod Deployment**: Pods are deployed to specified Namespaces (such as `app-ns`, `kube-system`)
4. **Profile Matching**: Fargate Profile selects qualified Pods based on configured rules
5. **Resource Allocation**: Selected Pods are allocated to Fargate Computing resources for execution

### The Role of Fargate Profile

Fargate Profile is the key configuration that determines which Pods should run on Fargate:

- **Namespace Selection**: Can specify specific Namespaces (such as `app-ns` in the diagram)
- **Pod Selector**: Precisely control which Pods use Fargate through Label Selectors
- **Resource Isolation**: Ensure different workloads run in appropriate computing environments

### Differences Between Fargate and Traditional EC2

- **No Node Management**: Fargate automatically manages underlying computing resources without manual EC2 instance configuration
- **Pay-per-Use**: Only pay for actually running Pods, no need to pay for idle nodes
- **Automatic Scaling**: Fargate automatically allocates appropriate computing resources based on Pod requirements
- **Higher Security**: Each Pod runs in an isolated computing environment, providing better isolation

### Use Cases and Considerations

**Scenarios suitable for Fargate:**
- Don't want to manage the complexity of EC2 instances
- Workloads with intermittent or unpredictable characteristics
- Applications requiring quick startup and shutdown
- Environments requiring high security isolation

**Considerations:**
- Fargate costs may be higher than EC2, especially for long-running workloads
- Some K8s features may have limitations on Fargate
- Proper Fargate Profile design is needed to ensure correct Pod scheduling