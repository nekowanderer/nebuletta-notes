# 關於 Kubernetes

[English](../en/01_about_kubernetes.md) | [繁體中文](../zh-tw/01_about_kubernetes.md) | [日本語](../ja/01_about_kubernetes.md) | [回到索引](../README.md)

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
代表你要部署的應用程式本身，可能是一個網站、API、服務等。

##### what to deploy
說明「要部署什麼」，也就是你的應用程式會被包裝成什麼形式來部署。在 k8s 的世界裡，要部署的東西就是 image。

### Image（docker / podman）
應用程式會被打包成映像檔（Image），常見的工具有 Docker 或 Podman。這個映像檔包含了應用程式執行所需的所有環境與依賴，確保在不同環境下都能一致運作。

### Deployment plan
部署計畫，描述「如何部署」以及「部署到哪裡」。這裡會規劃資源分配、網路設定、擴展策略等。

### Kubernetes
Kubernetes 是一個容器編排平台，負責自動化部署、擴展和管理容器化應用程式。它會根據 deployment plan 來執行部署。

##### compute
- 計算資源的分配與管理，Kubernetes 會用 deployment 物件來管理應用程式的副本數量與更新策略。
- Template: deployment

##### scaling（compute 的子項）
- 自動或手動調整應用程式的副本數量，以因應流量變化，確保服務穩定。
- HPA (Horizontal Pod Autoscaling)，這個是 k8s 跟普通容器技術差異最大的地方之一，普通的容器技術如 docker/podman 是無法根據流量進行增減的。

##### network
- 網路連線與服務暴露，Kubernetes 會用 service 物件來讓外部或內部流量能夠存取應用程式。
- Template: service

##### storage
- 儲存資源的管理，Kubernetes 會用 volume 物件來掛載持久化儲存空間給容器使用。
- Template: PVC (Persistent Volume Claim)

### Deployment
實際的部署行動，將應用程式與資源配置到指定的執行環境。

### Private cloud
所有伺服器或是硬體設備都自己買自己架，難度很高。

### Public cloud
公有雲平台，提供大規模、彈性且高可用的部署環境。常見的 Kubernetes 服務有：
- GCP GKE：Google Cloud Platform Kubernetes Engine
- AWS EKS：Amazon Web Services Elastic Kubernetes Service
- AWS ECS：Amazon Web Services Elastic Container Service（雖然不是原生 Kubernetes，但也是常見的容器服務）
- Azure AKS：Microsoft Azure Kubernetes Service

---
