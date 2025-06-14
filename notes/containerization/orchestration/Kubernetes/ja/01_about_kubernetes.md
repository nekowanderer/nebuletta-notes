# Kubernetesについて

[English](../en/01_about_kubernetes.md) | [繁體中文](../zh-tw/01_about_kubernetes.md) | [日本語](../ja/01_about_kubernetes.md) | [インデックスに戻る](../README.md)

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
デプロイしたいアプリケーション自体を指します。ウェブサイト、API、サービスなどが該当します。

##### what to deploy
「何をデプロイするか」を示します。アプリケーションがどのような形でパッケージ化されてデプロイされるかを説明します。k8sの世界では、デプロイ対象はイメージ（image）です。

### Image（docker / podman）
アプリケーションはDockerやPodmanなどのツールでイメージとしてパッケージ化されます。このイメージには、アプリケーションの実行に必要なすべての環境や依存関係が含まれており、異なる環境でも一貫して動作することが保証されます。

### Deployment plan
デプロイ計画は「どのようにデプロイするか」「どこにデプロイするか」を記述します。リソースの割り当て、ネットワーク設定、スケーリング戦略などが含まれます。

### Kubernetes
Kubernetesは、コンテナ化されたアプリケーションのデプロイ、自動スケーリング、管理を自動化するためのオーケストレーションプラットフォームです。デプロイメントプランに従ってデプロイを実行します。

##### compute
- コンピューティングリソースの割り当てと管理。Kubernetesはdeploymentオブジェクトを使ってアプリケーションのレプリカ数や更新戦略を管理します。
- テンプレート: deployment

##### scaling（computeの子項目）
- アプリケーションのレプリカ数を自動または手動で調整し、トラフィックの変化に対応してサービスの安定性を確保します。
- HPA（Horizontal Pod Autoscaling）は、k8sと従来のコンテナ技術との最大の違いの一つです。dockerやpodmanなどの従来のコンテナツールはトラフィックに応じたスケーリングができません。

##### network
- ネットワーク接続とサービス公開。Kubernetesはserviceオブジェクトを使って、内部または外部からアプリケーションへのアクセスを可能にします。
- テンプレート: service

##### storage
- ストレージリソースの管理。Kubernetesはvolumeオブジェクトを使って、コンテナに永続ストレージをマウントします。
- テンプレート: PVC（Persistent Volume Claim）

### Deployment
実際のデプロイ作業で、アプリケーションやリソースを指定された実行環境に割り当てます。

### Private cloud
すべてのサーバーやハードウェアを自社で購入・運用する形態です。難易度が非常に高いです。

### Public cloud
パブリッククラウドは、大規模で柔軟性が高く、高可用性のデプロイ環境を提供します。代表的なKubernetesサービスは以下の通りです：
- GCP GKE：Google Cloud Platform Kubernetes Engine
- AWS EKS：Amazon Web Services Elastic Kubernetes Service
- AWS ECS：Amazon Web Services Elastic Container Service（ネイティブKubernetesではありませんが、一般的なコンテナサービスです）
- Azure AKS：Microsoft Azure Kubernetes Service

--- 