# Kubernetes Computing Architecture Fundamentals

[English](../en/07_k8s_computing_architecture.md) | [繁體中文](../zh-tw/07_k8s_computing_architecture.md) | [日本語](../ja/07_k8s_computing_architecture.md) | [Back to Index](../README.md)

```
+----------------------------------------------------------------------------+
| Cluster                                                                    |
|                                                                            |
| +--------------------+   +--------------------+   +--------------------+   |
| |        Pod         |   |        Pod         |   |        Pod         |   |
| |  +-------------+   |   |  +-------------+   |   |  +-------------+   |   |
| |  |  Container  |   |   |  |  Container  |   |   |  |  Container  |   |   |
| |  |  Image [v2] |   |   |  |  Image [v2] |   |   |  |  Image [v1] |   |   |
| |  +-------------+   |   |  +-------------+   |   |  +-------------+   |   |
| +--------------------+   +--------------------+   +--------------------+   |
|           |                        |                        |              |
| +---------------------------------------------+   +-------------------+    |
| |                  ReplicaSet [v2]            |   |  ReplicaSet [v1]  |    |
| +---------------------------------------------+   +-------------------+    |
|           |                        |                        |              |
|           |                        |                        |              |
| +-----------------------------------------------------------------------+  |
| |                           Deployment [v2]                             |  |
| +-----------------------------------------------------------------------+  |
+----------------------------------------------------------------------------+

```

## Core Components Overview

#### Cluster
- The outermost box represents the entire Kubernetes cluster
- It is the foundational environment where all services run

#### Pod
- There are several Pods in the cluster
- Pods are the smallest deployable units that can be created and managed in Kubernetes
- Each Pod has its own IP address
- Can contain one or more containers

#### Container
- Applications running inside Pods
- Containers are instances created from Images

#### Image
- Templates used to create Containers
- The diagram shows two versions of Images: v1 and v2

#### ReplicaSet
- Ensures that a specified number of Pod replicas are running at any given time
- The diagram shows two ReplicaSets:
  - **ReplicaSet v1**: Manages Pods running Image v1
  - **ReplicaSet v2**: Manages Pods running Image v2
- This diagram illustrates the Zero-Downtime Rolling Update process, where normally all Pods should contain the same version of the image

#### Deployment
- A higher-level object used to manage ReplicaSets
- Through Deployments, you can describe the desired state of your application
  - For example: which version of Image to use, how many replicas are needed
- Deployments are responsible for creating and managing ReplicaSets to achieve this state

## Application Update Process

#### Update Process Overview
This diagram captures the transition moment when an application is being updated from version v1 to v2. The Deployment has already been updated to v2 configuration.

#### Update Steps
1. **Developer Updates Configuration**: Updates the Deployment configuration, changing the application's Image from v1 to v2
2. **Create New ReplicaSet**: The Deployment creates a new ReplicaSet v2
3. **Start New Pods**: ReplicaSet v2 begins creating new Pods using Image v2
4. **Gradual Replacement**: Meanwhile, the old ReplicaSet v1 gradually reduces the number of old Pods (Image v1) it manages

#### Current State
The diagram shows a moment in this process:
- 2 new v2 Pods are already running
- 1 old v1 Pod has not yet been terminated

#### Update Completion
Once all new v2 Pods are successfully running, the old ReplicaSet v1 will scale down the number of Pods it manages to 0, completing the entire update process. **This approach ensures that services remain uninterrupted during the update process.** 