# About K8s Namespace

[English](../en/24_about_k8s_namespace.md) | [繁體中文](../zh-tw/24_about_k8s_namespace.md) | [日本語](../ja/24_about_k8s_namespace.md) | [Back to Index](../README.md)

## What is a Namespace?

- A Namespace is a logical partition within a Kubernetes cluster that provides isolation and grouping management for resources (such as Pods, Services, Deployments, etc.).
- Resource names within each Namespace must be unique, but different Namespaces can have resources with the same name without affecting each other.

## Resource Hierarchy

<img src="../images/22_resource_level.jpg" width="600"/>

- A Cluster can contain multiple Namespaces.
- **Resources that can be placed in a Namespace**:
  - Service
  - Pod (container instances)
  - ReplicaSet
  - Deployment
  - HPA (Horizontal Pod Autoscaler)
  - PVC (PersistentVolumeClaim)
  - Ingress
- **Resources that cannot be placed in a Namespace (Cluster-level resources)**:
  - PV (PersistentVolume)
  - StorageClass
  - Node (Worker Node)
- Resources that "can be placed in a Namespace" are isolated from each other; resources that "cannot be placed in a Namespace" are shared across the entire Cluster.

## Relationship Between Nodes and Namespaces

- A Node (host) can simultaneously run Pods from multiple Namespaces.
- This means that while Pods logically belong to different Namespaces, they can all be scheduled to run on the same Node.
- This indicates that Nodes are "physical or virtual computing resources," while Namespaces are "logical partitions."
- Example:
  - A Worker Node might simultaneously run Pods from different Namespaces such as default, app001, app002, etc.
  - These Pods are logically isolated but physically share the same Node's resources.

## Separation Through Namespaces

- A Cluster can have multiple Namespaces, for example:
  - kube-system (system Namespace)
  - default (default Namespace)
  - app001, app002 (custom application Namespaces)
- Each Namespace can have its own Services, Pods, ReplicaSets, Deployments, PVCs, etc.
- Ingress can manage traffic routing across Namespaces.
- Namespaces allow different teams or applications to coexist in the same Cluster without interference, facilitating permission control and resource allocation.
- **By deleting a Namespace, you can delete all resources under that Namespace at once (such as Pods, Services, Deployments, PVCs, etc.), which is very convenient for cleaning up test environments or temporary projects.**
- **Note: Cluster-level resources (such as PV, StorageClass, Node) will not disappear when a Namespace is deleted.**

## Summary

- Namespace is the fundamental unit for resource isolation in Kubernetes clusters.
- Suitable for distinguishing different environments (such as dev, test, prod) or different projects and teams.
- PV, StorageClass, and Node belong to the Cluster level, while PVC belongs to the Namespace level.
- A Node can simultaneously run Pods from multiple Namespaces; Namespaces only provide logical isolation, while Nodes are physical or virtual computing resources.
- Proper use of Namespaces can improve management efficiency and security.