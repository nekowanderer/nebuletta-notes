# ECS, Fargate, Lambda, EKS Comparison and Sidecar Design Explanation

[English](05_aws_ecs_fargate_comparison.md) | [繁體中文](../zh-tw/05_aws_ecs_fargate_comparison.md) | [日本語](../ja/05_aws_ecs_fargate_comparison.md) | [Back to Index](../README.md)

## EC2 vs Fargate

| Aspect | EC2 | Fargate |
|--------|-----|---------|
| Nature | VM (Virtual Machine) | Serverless Container Runtime |
| Control | Self-managed VM (updates, monitoring, patching) | AWS manages infrastructure, you only handle container configuration |
| Cost Structure | May waste money on long-running instances | Pay only for actual usage time, more cost-effective |
| Maintenance | Need to patch OS, adjust resources, etc. | AWS manages automatically, developers focus on application logic |
| Suitable Scenarios | Long-running, special requirements | Task-based work, automatic scaling needs |

### Use Cases
- EC2 is suitable for:
  - Need complete environment control
  - Applications requiring special kernel modules
  - Long-running programs or those difficult to containerize
  - Like summoning a Bugbear in Lineage
- Fargate is suitable for:
  - Short-term tasks, event-triggered computing
  - Quick start, quick termination (like batch processing)
  - Teams that don't want to manage hosts, just deliver applications
  - No physical summoning unit (?)

---

## ECS on EC2 vs ECS on Fargate

| Metaphor | ECS on EC2 | ECS on Fargate |
|----------|------------|----------------|
| Summoning Nature | "You call them, and you need to prepare room and food" | "You call them, they find their own room and bring their own lunch" |
| Resource Management | You manage EC2 instances (CPU, RAM, Auto Scaling) | AWS automatically configures required resources, no EC2 management needed |
| Cost | Need to maintain summoned units (whether running tasks or not) | Pay only for task execution time (more like pay-per-use) |
| Scalability | Need to set up ASG or manual adjustment | Native elasticity, highly automated |
| Maintenance Responsibility | EC2 patching, resource pressure, capacity estimation all on you | AWS handles infrastructure, developers focus on application logic |
| Suitable Scenarios | Clear cost control, special environment requirements, long running time | Small teams, rapid development, scenarios with changing requirements |

---

## Fargate vs Lambda

| Aspect | Fargate | Lambda |
|--------|---------|--------|
| Summoning Nature | "Can handle entire tasks, can work long-term" | "Do one thing and run, like a precise assassin" |
| Startup Method | Can run long-term container tasks (background jobs, API, batch) | Usually event-triggered (API Gateway, S3, CloudWatch, etc.) |
| Duration | Can run from minutes to hours | Maximum 15 minutes (default cold start) |
| Control Flexibility | Docker-based, supports multiple languages and runtimes | Limited language support, needs AWS-supported runtime |
| Running Cost | Calculated by CPU and memory (based on configuration and time) | Billed by function runtime and memory (in milliseconds) |
| Suitable Scenarios | Stateful tasks or need more runtime control | Short and precise tasks like file conversion, notifications, image resize |

---

## Auto Scaling Group vs ECS Service Auto Scaling

| Name | What it Manages | Where it Applies | Adjustment Target |
|------|----------------|------------------|-------------------|
| Auto Scaling Group (ASG) | EC2 Count | ECS on EC2 | VM Count |
| ECS Service Auto Scaling | Task Count | ECS on Fargate / EC2 | Container Task Count |

### Detailed Explanation:
#### Auto Scaling Group (ASG)
- Used in EC2 (or ECS on EC2) environments
- Adjusts the number of virtual machines
- For example, if you set ASG min=2, max=10, and CPU pressure > 70%, it will automatically add an EC2 instance

#### Use Cases:
- When EC2 is primary and VMs need scaling
- In ECS on EC2, when containers can't run more tasks, ASG is needed to add EC2 instances

#### ECS Service Auto Scaling (for Fargate & EC2)
- Adjusts ECS task count, meaning container instance count
- You can set Target Tracking, e.g., maintain average CPU at 50%, it will automatically add tasks
- For Fargate, new tasks are automatically placed by AWS
- For EC2, you need to ensure EC2 VM has space, otherwise tasks will be pending

#### Use Cases:
- When you want container count to automatically scale
- Completely automated on Fargate, no need to manage underlying VMs
- On EC2, needs to work with ASG for complete elasticity

---

## ECS Task ≠ Container Instance

- Q: So, is ECS task equivalent to container instance?
- A: This statement needs careful interpretation, as ECS task ≠ container instance, although they may look similar in some contexts.

| Term | Description | Metaphor |
|------|-------------|----------|
| **ECS Task** | A running instance of "a group of containers" based on task definition.<br>Can run one or multiple containers. | 🧳 "Task Package": A small backpack containing 1 or more containers, ready to go when called |
| **Container Instance** | This term **only appears in ECS on EC2**, meaning an EC2 registered to ECS. | 🏠 "Container Host": A house that can hold many ECS tasks |
| **Fargate has no Container Instance** | Fargate is serverless, no EC2. Tasks are directly placed by AWS in managed locations. | Cloud Apartment: AWS automatically assigns each task an independent room, you don't need to worry about address or cleaning fees |

#### Differences
| Question | Fargate | ECS on EC2 |
|----------|---------|------------|
| Has Container Instance? | ❌ No | ✅ Yes |
| How many containers per task? | Depends on task definition, usually 1~N | Same |
| Who finds space for tasks? | AWS automatically | ECS scheduler finds space in container instances |

---

## Sidecar Pattern Example: Microservice + OTEL + Fluent Bit

#### Why might an ECS Task contain multiple containers?
Because:
> An ECS Task is launched based on Task Definition, which can contain multiple containers.

This design supports the sidecar pattern, allowing different containers to share:
  - Same network namespace
  - Same storage space (Volumes)
  - And be scheduled and managed together

| Container | Role | Function Description |
|-----------|------|---------------------|
| 🧩 microservice | Main container | Actual business logic API or service |
| 📈 OTEL Collector | Metrics/Traces sidecar | Collects and forwards OpenTelemetry format metrics & tracing (usually OTLP over gRPC or HTTP) |
| 📤 Fluent Bit | Logs sidecar | Collects stdout logs and forwards to log sinks like CloudWatch, S3, Elasticsearch |

This architecture separates logs, metrics, and traces while deploying in the same task, with benefits:
- Decouples observability from main code:
  - No need to hardcode log/trace/export logic in microservice
  - OTEL Collector and Fluent Bit can be maintained by infra team
- High reusability:
  - Multiple microservices can use the same sidecar pattern
  - Just adjust config file for different needs
- Low coupling, high deployment flexibility:
  - Easy to migrate if you want to move OTEL Collector to a side service
- Supports shared volumes, port communication, dependsOn settings, forming independent task deployment units

#### Common uses for sidecar containers

| Type | Function |
|------|----------|
| Logging | Send logs to S3, CloudWatch, Elasticsearch |
| Metrics | Prometheus exporter, Datadog agent |
| Proxy | Envoy, NGINX, Linkerd as reverse proxy / service mesh |
| Security | Like Istio Citadel providing certificates, auto-updating certificate agents |
| Init container (not in ECS) | Only available in K8s, ECS doesn't have this mechanism |

---

## ECS Doesn't Support initContainer and preStop

ECS tasks launch containers together based on task definition, but it:

| Feature | ECS Support |
|---------|-------------|
| Container startup order (`dependsOn`) | ✅ Basic support |
| Init container (one-time initialization) | ❌ No |
| preStop hook / graceful shutdown | ❌ No built-in support |
| Sidecar restart policy separation | ❌ No |
| Kill entire task if container healthcheck fails | ✅ Configurable, but crude |

So ECS's design is:
> "All containers are bound in one task, live and die together."

Unlike K8s which can finely define: "This container is init and runs once, this container's failure doesn't affect the main container, this one needs to wait for others to be ready."

---

## For Fine-grained Lifecycle Control, Consider AWS EKS

#### When to use EKS
| When you need these | Consider EKS |
|---------------------|--------------|
| Complex dependencies between containers | ✅ |
| Multi-stage initialization process | ✅ |
| Enhanced observability, monitoring, or network policies | ✅ |
| Want to integrate Istio, Linkerd, etc. Service Mesh | ✅ |
| Want to customize container lifecycle hook behavior | ✅ |

#### Lifecycle
| Feature | EKS Support |
|---------|-------------|
| initContainer | ✅ |
| preStop hook | ✅ |
| Readiness / Liveness probe | ✅ |
| terminationGracePeriod | ✅ |
| Service Mesh / Sidecar decoupling | ✅ |

For strict container lifecycle control and flexible architecture, consider using AWS EKS (Kubernetes managed service).

## ECS vs EKS

| Aspect | ECS | EKS |
|--------|-----|-----|
| Operation Difficulty | Simple (can be seen as "K8s-lite") | More complex, need to learn Kubernetes |
| Cost | Controllable, simpler resource usage | Higher management cost, may pay more for complexity |
| Scaling Flexibility | Medium | High, suitable for large systems |
| Community & Toolchain | Mainly AWS ecosystem | Worldwide K8s ecosystem support | 