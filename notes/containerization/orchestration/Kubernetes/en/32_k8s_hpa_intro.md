# Introduction to Kubernetes HPA (Horizontal Pod Autoscaling)

[English](../en/32_k8s_hpa_intro.md) | [繁體中文](../zh-tw/32_k8s_hpa_intro.md) | [日本語](../ja/32_k8s_hpa_intro.md) | [Back to Index](../README.md)

Kubernetes HPA automatically adjusts the number of Pods based on resource utilization (such as CPU usage), ensuring applications can scale elastically up or down. The workflow is as follows:

- HPA continuously monitors the CPU usage of all Pods within a Deployment.
- These resource usage data are collected by the Metric Server. The Metric Server periodically gathers CPU (or memory) usage information from each Pod and provides this data to HPA.
- After obtaining this real-time monitoring data, HPA determines whether to adjust the number of Pods in the ReplicaSet based on predefined thresholds (such as target CPU usage) (for example, increasing or decreasing from 3 Pods).
- When load increases, HPA automatically scales up the number of Pods; when load decreases, it automatically scales down, improving resource utilization and service stability.

## Key Component Descriptions
- **Cluster**: Kubernetes cluster environment.
- **Deployment**: Manages application deployment and upgrades.
- **ReplicaSet**: Ensures a specified number of Pods continue running.
- **Pod**: The smallest unit where applications run, containing one or more Containers.
- **Metric Server**: Responsible for collecting and providing real-time resource usage data from Pods in the cluster for HPA reference.
- **HPA**: Automatically adjusts the number of Pods based on data provided by the Metric Server.

## Comparison Between Metric Server and Prometheus

- **Metric Server**: Kubernetes' built-in lightweight resource monitoring component, primarily providing real-time basic metrics like CPU and memory for HPA and other internal components. Simple installation, suitable for scenarios requiring only basic autoscaling functionality.
- **Prometheus**: A more powerful monitoring system that can collect more detailed and diverse metrics (including custom application metrics), store data long-term, and support complex queries and alerting. By installing solutions like kube-prometheus-stack or custom Adapters, HPA can directly obtain metrics from Prometheus for more advanced autoscaling.

**Summary**:
If you only need basic HPA functionality, using Metric Server is sufficient; for more advanced monitoring and autoscaling requirements, consider replacing Metric Server with Prometheus and using custom metrics for flexible adjustments.

## Pull and Push Models in Metrics Server

Kubernetes' built-in Metric Server and mainstream monitoring systems like Prometheus primarily adopt the "Pull" model. This means Prometheus actively sends periodic requests to each Pod or service's `/metrics` HTTP endpoint to collect real-time monitoring data. The advantages of this approach include:
- The monitoring system centrally manages collection frequency and targets, making it easy to manage and adjust.
- Can immediately detect if target services are offline (because data cannot be pulled).
- Works with service discovery (like Kubernetes Service Discovery) to automatically track dynamically changing targets.

Some other monitoring systems or databases (like ClickHouse, Graphite) support the "Push" model. In this mode, applications actively send monitoring data to specified metrics servers via HTTP, UDP, and other protocols. The advantages of this approach include:
- Suitable for short-lived tasks (like batch jobs), which can actively push data once when tasks complete.
- Easy to simultaneously push data to multiple receivers.

### Specific Examples
- **Prometheus**: Default "Pull" model, actively fetching target metrics endpoints periodically. For push support, can use Pushgateway as an intermediary.
- **ClickHouse**: Supports Prometheus remote-write (Push) protocol, allowing applications to directly push metrics to ClickHouse via HTTP.
- **Graphite**: Native support for "Push" model, where applications directly send data to Graphite.

**Summary**:
- **Pull Model**: Prometheus, Kubernetes Metric Server
- **Push Model**: ClickHouse (remote-write), Graphite
- In practice, both models have their advantages and disadvantages, and the choice can be flexibly adjusted based on application scenarios and infrastructure requirements.