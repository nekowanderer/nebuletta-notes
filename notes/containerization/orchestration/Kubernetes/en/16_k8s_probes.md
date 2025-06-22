# Various Probes in Kubernetes

[English](../en/16_k8s_probes.md) | [繁體中文](../zh-tw/16_k8s_probes.md) | [日本語](../ja/16_k8s_probes.md) | [Back to Index](../README.md)

In Kubernetes, probes are health checks performed by the kubelet on containers periodically to ensure application reliability. Based on different inspection purposes, probes are mainly divided into three types.

### Probe Overview

*   **`StartupProbe` (Startup Probe)**: Determines whether the application inside the container has "started up completely". This is particularly useful for slow-starting applications, preventing them from being incorrectly terminated by other probes before they are ready.
*   **`LivenessProbe` (Liveness Probe)**: Determines whether the application is still "running and healthy". If the probe fails, the kubelet will restart the container, primarily used for "self-monitoring".
*   **`ReadinessProbe` (Readiness Probe)**: Determines whether the application is "ready to receive traffic". If the probe fails, Kubernetes will pause sending requests to it. It is primarily used for "monitoring external dependencies", such as database connections, other APIs, etc.

### Probe Flow Diagram

This diagram depicts the complete lifecycle from Pod startup to when various probes begin operating:

```text
+-----------------+
| Pod             |
| Initialization  |
+-----------------+
        |
        v
+-----------------------------------------------------------------------+
| StartupProbe                                                          |
| (Check if application has started)                                    |
|                                                                       |
| [Success] ---------------------> Start LivenessProbe & ReadinessProbe |
| [Failure] ---------------------> [Restart Pod]                        |
+-----------------------------------------------------------------------+
        |
        | (After StartupProbe success)
        |
        +----------------------------------------------------------+
        |                                                          |
        v                                                          v
+----------------------------+                     +------------------------------+
| LivenessProbe              |                     | ReadinessProbe               |
| (Monitor if app is frozen) |                     | (Monitor external deps)      |
|                            |                     |                              |
| [Failure] -> [Restart Pod] |                     | [Failure] -> [Mark Unready]  |
|                            |                     |              (Stop traffic)  |
| [Success] -> (Continue)    |                     |              |               |
|                            |                     |              v               |
|                            |                     | [Success] -> [Mark Ready]    |
|                            |                     |              (Resume traffic)|
+----------------------------+                     +------------------------------+
```

### Characteristics of Each Probe

| Feature | StartupProbe (Startup Probe) | LivenessProbe (Liveness Probe) | ReadinessProbe (Readiness Probe) |
| :--- | :--- | :--- | :--- |
| **Purpose** | Primarily for applications requiring longer startup time, preventing accidental termination by `LivenessProbe` before startup completion. | Ensure the application is in a running state, commonly used to resolve deadlock issues. | Determine if the application is "ready" to receive traffic, commonly used for dependency checking during service startup. |
| **Monitoring Target** | Application itself | Application itself | External dependencies of the application<br/>(such as: DB connections, other APIs) |
| **Failure Behavior** | Kubelet restarts Pod | Kubelet restarts container | Endpoint Controller removes Pod from Service, pausing traffic. |
| **Success Behavior** | No longer executes, hands over to subsequent probes. | Container continues running with periodic monitoring. | Pod is added back to Service, starts (or resumes) receiving traffic. |

### Recommendations and Configuration

When designing probes, you can follow these best practices:

#### When to Use Which Probe?

1.  **`StartupProbe`**: When your application's startup time is very long, exceeding `LivenessProbe`'s `failureThreshold * periodSeconds`, you should use `StartupProbe`.
2.  **`LivenessProbe`**: Almost all long-running services should be configured. It can catch application crashes or deadlock issues. However, the probe logic should be as lightweight as possible to avoid additional burden on the application.
3.  **`ReadinessProbe`**: When your application depends on external systems (such as databases, caches, other APIs), or needs to perform some initialization work after startup before providing services, you must use `ReadinessProbe`. This ensures traffic only comes in when fully ready, achieving graceful service startup and rolling updates.

#### Differences in Probe Endpoint Design

A common mistake is having Liveness Probe and Readiness Probe use the same health check endpoint. This is a bad practice because their concerns are different:

*   **Liveness Endpoint (`/healthz`, `/livez`)**: Should only report the application's own status and should not check external dependencies. If Liveness Probe fails due to database temporary unavailability, the Pod will be unnecessarily restarted, potentially causing a cascading restart storm.
*   **Readiness Endpoint (`/readyz`, `/ready`)**: Should check everything needed for the application to provide complete service, including connections to backend databases or other services. When backend services have issues, Readiness Probe failure will temporarily take the Pod offline, automatically bringing it back online when backend services recover, which is a more graceful handling approach.

#### Configuration Example

This is a Pod configuration example that combines all three types of probes:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: my-app-container
    image: my-app:1.0
    ports:
    - containerPort: 8080
    
    # StartupProbe: Give the application up to 5 minutes (30 * 10s) startup time
    startupProbe:
      httpGet:
        path: /api/health/startup
        port: 8080
      failureThreshold: 30
      periodSeconds: 10

    # LivenessProbe: After startup, check liveness every 15 seconds
    livenessProbe:
      httpGet:
        path: /api/health/liveness
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 15
      timeoutSeconds: 2
      failureThreshold: 3

    # ReadinessProbe: After startup, check service readiness every 20 seconds
    readinessProbe:
      httpGet:
        path: /api/health/readiness
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 20
      timeoutSeconds: 5
      failureThreshold: 3
```