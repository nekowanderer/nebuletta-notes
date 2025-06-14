# About Kubernetes

[English](../en/01_about_kubernetes.md) | [繁體中文](../zh-tw/01_about_kubernetes.md) | [日本語](../ja/01_about_kubernetes.md) | [Back to Index](../README.md)

```
+-------------------+   |     +------------------------+   |     +-------------------+
|       App         |   |     |    Deployment plan     |   |     |    Deployment     |
+-------------------+   |     +------------------------+   |     +-------------------+
| what to deploy    |   |     | how to deploy          |   |     | actual deploy     |
+-------------------+   |     | where to deploy        |   |     | actual resources  |
                        |     +------------------------+   |     +-------------------+
                        |                                  |                          
                        |                                  |                          
+-------------------+   |     +------------------------+   |     +-------------------+
|      Image        |   | ->  |      Kubernetes        |   | ->  |   Private cloud   |
+-------------------+   |     +------------------------+   |     +-------------------+
| docker / podman   |   |     | compute (deployment)   |   |                           
+-------------------+   |     |   └─ scaling (HPA)     |   |                                                   
                        |     | network (service)      |   |     +-------------------+
                        |     | storage (PVC)          |   |     |   Public cloud    |
                        |     +------------------------+   |     +-------------------+
                        |                                  |     |    GCP GKE        |
                        |                                  |     |    AWS EKS        |
                        |                                  |     |    AWS ECS        |
                        |                                  |     |    Azure AKS      |
                        |                                  |     +-------------------+
```

---

### App
This represents the application you want to deploy, such as a website, API, or service.

##### what to deploy
Describes "what to deploy"—that is, in what form your application will be packaged for deployment. In the world of k8s, what you deploy is the image.

### Image (docker / podman)
The application is packaged into an image, commonly using Docker or Podman. This image contains all the environment and dependencies required to run the application, ensuring consistent operation across different environments.

### Deployment plan
The deployment plan describes "how to deploy" and "where to deploy." This includes resource allocation, network configuration, scaling strategies, and more.

### Kubernetes
Kubernetes is a container orchestration platform responsible for automating the deployment, scaling, and management of containerized applications. It executes deployments according to the deployment plan.

##### compute
- Allocation and management of computing resources. Kubernetes uses the deployment object to manage the number of replicas and update strategies for applications.
- Template: deployment

##### scaling (child of compute)
- Automatically or manually adjusts the number of application replicas to handle traffic changes and ensure service stability.
- HPA (Horizontal Pod Autoscaling) is one of the biggest differences between k8s and traditional container technologies. Traditional container tools like docker/podman cannot scale based on traffic.

##### network
- Network connectivity and service exposure. Kubernetes uses the service object to allow internal or external traffic to access the application.
- Template: service

##### storage
- Management of storage resources. Kubernetes uses the volume object to mount persistent storage for containers.
- Template: PVC (Persistent Volume Claim)

### Deployment
The actual deployment action, assigning the application and resources to the specified execution environment.

### Private cloud
All servers or hardware are purchased and managed by yourself. This is very challenging.

### Public cloud
Public cloud platforms provide large-scale, flexible, and highly available deployment environments. Common Kubernetes services include:
- GCP GKE: Google Cloud Platform Kubernetes Engine
- AWS EKS: Amazon Web Services Elastic Kubernetes Service
- AWS ECS: Amazon Web Services Elastic Container Service (not native Kubernetes, but a common container service)
- Azure AKS: Microsoft Azure Kubernetes Service

--- 