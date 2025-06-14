# What Problems Does K8S Solve?

[English](../en/02_what_k8s_solved.md) | [繁體中文](../zh-tw/02_what_k8s_solved.md) | [日本語](../ja/02_what_k8s_solved.md) | [Back to Index](../README.md)

### 1. Dynamic Scaling
K8S provides auto-scaling capabilities, automatically adjusting the number of application replicas based on traffic or computing demands. When traffic increases, it automatically adds more Pods; when traffic decreases, it reduces them, ensuring optimal resource utilization and cost savings.

### 2. Self Healing
K8S continuously monitors the health of applications. If a Pod becomes unhealthy or fails, K8S will automatically restart or recreate a new Pod to replace the faulty one, ensuring service availability and minimizing manual intervention.

### 3. Zero-Downtime Rolling Update
K8S supports rolling updates, allowing old versions of applications to be gradually replaced by new versions without service interruption. Users experience no downtime, and the update process is smooth and safe.

### 4. Zero-Downtime Rollback
If issues are detected after deploying a new version, K8S can quickly roll back to a previously stable version, also without causing service downtime. This makes deployments more flexible and reduces the risk of failed updates. 