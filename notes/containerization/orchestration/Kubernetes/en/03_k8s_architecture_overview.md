# Overview of Kubernetes Architecture

[English](../en/03_k8s_architecture_overview.md) | [繁體中文](../zh-tw/03_k8s_architecture_overview.md) | [日本語](../ja/03_k8s_architecture_overview.md) | [Back to Index](../README.md)

---

## Kubernetes Architecture Diagram

```
+--------------------------- Cluster ----------------------------+
|   +-------------------+                +-------------------+   |
|   |    Master Node    |                |    Worker Node    |   |
|   |  +-------------+  |                |  +-------------+  |   |
|   |  | API Server  |  |                |  | Kubelet     |  |   |
|   |  |             |  |                |  | Service     |  |   |
|   |  +-------------+  |                |  +-------------+  |   |
|   |  | Cluster     |  |    control     |  | Kube Proxy  |  |   |
|   |  | Controller  |  | <============> |  | Service     |  |   |
|   |  | Manager     |  |  communicate   |  +-------------+  |   |
|   |  +-------------+  |                |  | Container   |  |   |
|   |  | Scheduler   |  |                |  | Runtime     |  |   |
|   |  +-------------+  |                |  +-------------+  |   |
|   |  | Cluster     |  |                |                   |   |
|   |  | State       |  |                |                   |   |
|   |  | Store       |  |                |                   |   |
|   |  +-------------+  |                |                   |   |
|   +-------------------+                +-------------------+   |
+----------------------------------------------------------------+
```

---

## About Master Node

The Master Node is the "brain" of the Kubernetes cluster, responsible for overall management, scheduling, and state maintenance. Main components include:

- **API Server**:
  - The communication hub of the cluster; all commands (CLI, YAML) go through here first.
  - Provides REST API, responsible for authentication, authorization, data validation, and scheduling.
- **Cluster Controller Manager**:
  - Manages various controllers (nodes, replicas, endpoints, etc.) to ensure the cluster state matches expectations.
- **Scheduler**:
  - Assigns newly created Pods to appropriate Worker Nodes.
- **Cluster State Store (etcd)**:
  - Distributed key-value database storing the entire cluster state.
- **Master Replicate Node**:
  - Provides high availability; automatically switches over if the main node fails.

---

## About Worker Node

Worker Nodes are where application containers (Pods) actually run. Main components include:

- **Kubelet Service**:
  - Communicates with the Master and manages the lifecycle of local Pods.
- **Kube Proxy Service**:
  - Handles network proxying and load balancing, enabling services to be accessed across nodes.
- **Container Runtime**:
  - The software that actually runs containers (e.g., Docker, Podman).

---

## Pod and Container Registry Explanation

In Kubernetes, a Pod is the smallest deployable unit. A Pod can contain one or more containers. Each container is started from an image, and these images are stored in a Container Registry, which can be local or cloud-based (such as DockerHub, AWS ECR, GCP GAR, Azure ACR, etc.).

When you deploy a Pod in a K8S cluster, Kubernetes will automatically pull the corresponding image from the specified registry and start the container on a Worker Node.

### Pod and Container Registry Relationship Diagram

```
            +--------- Cluster ---------+
            |                           |
            |   +-------------------+   |
            |   |        Pod        |   |
            |   |  +-------------+  |   |
            |   |  | Container   |  |   |
            |   |  | +---------+ |  |   |
            |   |  | | Image   | |  |   |
            |   |  | +---------+ |  |   |
            |   |  +------|------+  |   |
            |   +---------|---------+   |
            |             |             |
            +-------------|-------------+
                          | Pull image
                          v
+---------------- Container Registry ---------------+
| Local | DockerHub | AWS ECR | GCP GAR | Azure ACR |
+---------------------------------------------------+
```
- **Pod**: The smallest operational unit in K8S, usually corresponding to an application service.
- **Container**: The actual container running inside the Pod, started from an image.
- **Image**: A snapshot of the application runtime environment, including code, dependencies, etc.
- **Container Registry**: Centralized place to manage images, can be local or cloud-based.
- **Automatic Pull**: K8S will automatically pull images from the registry according to Pod settings.

---

## Relationship between Pod, Node, and Resource Provider

Pods in a Kubernetes cluster are assigned to different Worker Nodes for execution. These Worker Nodes actually run on various computing resources (Resource Providers), such as local machines, AWS EC2, GCP Compute Engine, Azure VM, etc.

This design allows K8S to flexibly choose the underlying computing platform, whether on-premises or in the cloud.

```
            +---------- Kubernetes Cluster ----------+
            |                                        |
            |   +-----+      +-------------------+   |
            |   | Pod |----> |                   |   |
            |   +-----+      |    Worker Node    |   |
            |   +-----+      |                   |   |
            |   | Pod |----> |                   |   |
            |   +-----+      +-------------------+   |
            |   +-----+      +-------------------+   |
            |   | Pod |----> |                   |   |
            |   +-----+      |    Worker Node    |   |
            |                |                   |   |
            |                +---------|---------+   |
            +--------------------------|-------------+
                                       |
                                       v
       +--------- Computing Resource Provider -----------+
       | Local | AWS EC2 | GCP Compute Engine | Azure VM |
       +-------------------------------------------------+
```
- **Worker Node**: The computing node in a K8S cluster that actually runs Pods.
- **Resource Provider**: The platform providing computing resources, can be local servers or VMs from major cloud providers.
- **Flexible Deployment**: K8S allows you to manage resources from multiple sources, enabling hybrid or multi-cloud deployments.

---

## Additional Notes

- **High Availability Design**:
  - Master Nodes usually have multiple replicas to avoid single points of failure.
- **Communication**:
  - Users can interact with the API Server via CLI or YAML files.
  - Master and Worker Nodes continuously communicate to ensure cluster state consistency.
- **Scalability**:
  - Worker Nodes can be scaled up or down at any time to handle different workloads.
- **Support for Multiple Container Runtimes**:
  - Such as Docker, Podman, etc.
- **Kubernetes Architecture Philosophy**:
  - Layered and modular design for easier management and maintenance.

--- 