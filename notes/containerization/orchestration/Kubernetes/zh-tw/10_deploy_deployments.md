# Kubernetes Deployment 部署實作

[English](../en/10_deploy_deployments.md) | [繁體中文](../zh-tw/10_deploy_deployments.md) | [日本語](../ja/10_deploy_deployments.md) | [回到索引](../README.md)

## 前置準備

#### 啟動 Docker 服務
```bash
$ sudo service docker start
$ docker ps
```

#### 建立 Minikube 叢集
```bash
$ minikube start --driver docker
$ minikube status
```

## 建立 Deployment

#### 建立 Deployment 定義檔
建立 `simple-deployment.yaml` 檔案：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app-pod
  template:
    metadata:
      labels:
        app: app-pod
    spec:
      containers:
      - name: app-container
        image: uopsdod/k8s-hostname-amd64-beta:v1
        ports: 
        - containerPort: 80
```

#### 部署 Deployment
```bash
# 部署 Deployment
$ kubectl apply -f simple-deployment.yaml

# 查看 Deployment 狀態
$ kubectl get deployments
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
app-deployment   3/3     3            3           19s

# 查看 ReplicaSet 狀態
$ kubectl get rs
NAME                        DESIRED   CURRENT   READY   AGE
app-deployment-8687d78656   3         3         3       28s

# 查看 Pod 狀態
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-8687d78656-5bf82   1/1     Running   0          31s
app-deployment-8687d78656-l4ctt   1/1     Running   0          31s
app-deployment-8687d78656-lv7wl   1/1     Running   0          31s

# 查看所有資源
$ kubectl get all
AME                                  READY   STATUS    RESTARTS   AGE
pod/app-deployment-8687d78656-5bf82   1/1     Running   0          37s
pod/app-deployment-8687d78656-l4ctt   1/1     Running   0          37s
pod/app-deployment-8687d78656-lv7wl   1/1     Running   0          37s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   51m

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/app-deployment   3/3     3            3           37s

NAME                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/app-deployment-8687d78656   3         3         3       37s
```

## 測試 Deployment 狀態維持機制

#### 模擬 Pod 故障
```bash
# 刪除所有 Pod，測試 Deployment 的自動重建功能
$ kubectl delete pods --all
pod "app-deployment-8687d78656-5bf82" deleted
pod "app-deployment-8687d78656-l4ctt" deleted
pod "app-deployment-8687d78656-lv7wl" deleted

# 觀察 Deployment 自動建立新的 Pod
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-8687d78656-2hhtc   1/1     Running   0          37s
app-deployment-8687d78656-4hcvv   1/1     Running   0          37s
app-deployment-8687d78656-4xfcd   1/1     Running   0          37s
```

#### 測試 ReplicaSet 刪除
```bash
# 刪除所有 ReplicaSet
$ kubectl delete rs --all
replicaset.apps "app-deployment-8687d78656" deleted

# 觀察 Deployment 自動建立新的 ReplicaSet
$ kubectl get rs
NAME                        DESIRED   CURRENT   READY   AGE
app-deployment-8687d78656   3         3         3       32s

# 確認 Pod 仍然存在
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-8687d78656-6hzmm   1/1     Running   0          78s
app-deployment-8687d78656-865xb   1/1     Running   0          78s
app-deployment-8687d78656-qm75q   1/1     Running   0          78s
```

## 清理資源
```bash
# 刪除所有 Deployment
$ kubectl delete deployments --all
```

## 說明

- **Deployment**: 提供 Pod 和 ReplicaSet 的宣告式更新，是 Kubernetes 中最常用的工作負載資源
- **replicas: 3**: 指定要維持的 Pod 副本數量
- **selector**: 定義如何識別屬於此 Deployment 的 Pod
- **template**: 定義 Pod 的規格，包含容器映像、端口等設定
- **自我修復**: 當 Pod 被刪除或故障時，Deployment 會自動建立新的 Pod 來維持指定的副本數量
- **kubectl get deployments**: 查看 Deployment 狀態，顯示準備就緒的副本數、最新版本副本數等資訊

## 重要概念

- **Deployment vs ReplicaSet**: Deployment 管理 ReplicaSet，而 ReplicaSet 管理 Pod，形成層級化的管理結構
- **期望狀態 vs 實際狀態**: Deployment 會持續監控並確保實際運行的 Pod 數量符合期望的副本數
- **標籤選擇器**: 使用 `matchLabels` 來識別和管理屬於此 Deployment 的 Pod
- **Pod 模板**: 定義新建立的 Pod 應該具備的規格和設定
- **版本控制**: Deployment 支援滾動更新和版本回滾功能（進階功能）
- **資源層級**: Deployment > ReplicaSet > Pod，刪除上層資源會影響下層資源


## 關於 Self-healing

- Pod 層級：
  - Deployment / ReplicaSet / StatefulSet 這些控制器會維持期望的副本數。如果某個 Pod crash 掉或被刪除，控制器會自動建立新的 Pod 來補足。這就是 K8S 常說的「heal pods」。

- Node 層級：
  - Kubernetes 本身不會修復或重建 Node。如果一個 Node 掛了（硬體壞掉、網路斷線或 VM 被刪掉），K8S 只會把這個 Node 標記為 NotReady，並經過一段時間後（由 node-controller 判斷），將該 Node 上的 Pod 標記為失敗，然後再重新在其他可用的 Node 上排程新的 Pod。

> ⚠️ 換句話說，K8S 只能做到「pod rescheduling」，但不能「heal node」或讓 node 自己復活。

#### 那 Node 要怎麼 heal？

- 這通常要靠外部的基礎設施層來處理，例如：
  - 雲端環境：
    - AWS EKS → 搭配 Auto Scaling Group (ASG) + Cluster Autoscaler，自動替換壞掉的 EC2 node。
    - GKE / AKS → 內建 node auto-repair 功能。
  - On-premise 環境：
    - 可能要用 MetalLB + Cluster Autoscaler 或額外的監控/自動化工具（例如 Ansible, Terraform, 或硬體管理系統）來做。

- 總結：
  - K8S 會 heal Pods，但 Node 的「healing」要靠雲端平台或額外的 node 管理工具來完成。

## 關於 Rolling update
Deployment 的 rolling update 流程

1. 使用者下指令 (kubectl apply -f ...)
  - 更新 Deployment 的規格 (例如 image: v2)。
  - 這個新的期望狀態會被寫進 etcd（K8S 的唯一狀態儲存）。
2. Deployment Controller 發現差異
  - 它會從 etcd 讀到「期望狀態」跟「實際狀態」不同。
  - 比如：目前 nginx:v1 的 ReplicaSet 有 5 個 Pod，期望是 nginx:v2。
3. 生成新的 ReplicaSet
  - Controller 建立一個新的 ReplicaSet（對應 nginx:v2）。
  - 舊的 ReplicaSet (nginx:v1) 不會馬上刪掉，而是逐步 scale down。
4. 滾動更新 (RollingUpdate)
  - Controller 根據 Deployment 的策略 (spec.strategy.rollingUpdate) 去控制：
    - maxUnavailable：一次可以有多少 Pod 不可用。
    - maxSurge：一次最多可以多開多少額外 Pod。
  - 實際上就是「同時 scale up 新 RS，scale down 舊 RS」，直到數量達標。
5. 狀態紀錄
  - 每個 Deployment 的 狀態 (status.conditions、status.replicas、status.updatedReplicas …) 都會更新並存到 etcd。
  - 這樣 kubectl rollout status deployment/... 就能查出目前更新進度。

小結：
  - etcd 是唯一真實來源 (source of truth)，Deployment 的 spec 和更新進度全都記錄在 etcd。
  - Deployment Controller 是執行者，負責根據 etcd 裡的「期望狀態」去做 Pod 的替換，並把過程狀態回寫 etcd。
