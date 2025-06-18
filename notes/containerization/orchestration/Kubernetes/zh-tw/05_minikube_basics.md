# MiniKube 安裝與使用指南

[English](../en/05_minikube_basics.md) | [繁體中文](../zh-tw/05_minikube_basics.md) | [日本語](../ja/05_minikube_basics.md) | [回到索引](../README.md)

## 目錄
1. [環境準備](#環境準備)
2. [安裝 Docker](#安裝-docker)
3. [安裝 MiniKube](#安裝-minikube)
4. [建立與管理 Cluster](#建立與管理-cluster)
5. [部署應用程式](#部署應用程式)
6. [服務管理](#服務管理)
7. [網路連線測試](#網路連線測試)
8. [清理資源](#清理資源)

---

## 安裝 Docker

#### 1. 安裝 Docker 套件
```bash
$ sudo yum install docker -y
```

#### 2. 設定使用者權限
```bash
# 將當前使用者加入 docker 群組
$ sudo usermod -aG docker $USER && newgrp docker
```

#### 3. 啟動 Docker 服務
```bash
$ sudo service docker start
```

#### 4. 驗證 Docker 安裝
```bash
$ docker ps
```

---

## 安裝 MiniKube

#### 1. 到使用者家目錄內，下載 MiniKube 執行檔
```bash
$ wget https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
```

#### 2. 設定執行權限並安裝
```bash
$ chmod +x minikube-linux-amd64
$ sudo mv minikube-linux-amd64 /usr/local/bin/minikube
```

#### 3. 驗證安裝
```bash
$ minikube version
minikube version: v1.36.0

$ minikube status
Profile "minikube" not found. Run "minikube profile list" to view all profiles.
To start a cluster, run: "minikube start"
```

---

## 建立與管理 Cluster

#### 1. 啟動 MiniKube Cluster
透過 driver 選項決定要在哪裡啟動運算資源，這邊可以想成 master node/worker node，使用 minikube 的時候這兩種 node 基本上都在同一個地方
```bash
$ minikube start --driver docker
```

#### 2. 檢查 Cluster 狀態
```bash
$ docker ps
CONTAINER ID   IMAGE                                 COMMAND                  CREATED         STATUS         PORTS                                 NAMES
d86c55dddc18   gcr.io/k8s-minikube/kicbase:v0.0.47   "/usr/local/bin/entr…"   4 minutes ago   Up 4 minutes   127.0.0.1:32777->22/tcp, 127.0.0.1:32776->2376/tcp, 127.0.0.1:32775->5000/tcp, 127.0.0.1:32774->8443/tcp, 127.0.0.1:32773->32443/tcp   minikube
```
這表示啟動了一個叫做 minikube 的 docker container，然後 K8S 就運作在這個 container 裡面，所以這個 container 就是目前的 master node (worker node 也是)

為了要維持 cluster 的運作，這時候就已經會有一些現行的 pod 在運作著，可以用以下指令查詢：
```bash
# kubectl 是可以讓我們跟 K8S cluster 溝通的主要指令
# -A 表示顯示所有 pod
$ minikube kubectl -- get pods -A
AMESPACE     NAME                               READY   STATUS    RESTARTS        AGE
kube-system   coredns-674b8bbfcf-9jcq5           1/1     Running   0               8m11s
kube-system   etcd-minikube                      1/1     Running   0               8m16s
kube-system   kube-apiserver-minikube            1/1     Running   0               8m16s
kube-system   kube-controller-manager-minikube   1/1     Running   0               8m18s
kube-system   kube-proxy-6x55d                   1/1     Running   0               8m12s
kube-system   kube-scheduler-minikube            1/1     Running   0               8m18s
kube-system   storage-provisioner                1/1     Running   1 (7m41s ago)   8m14s
```

#### 3. 設定 kubectl 快捷鍵
```bash
# 編輯 bashrc 檔案
vi ~/.bashrc

# 加入以下別名設定
alias kubectl='minikube kubectl -- '

# 重新載入設定
source ~/.bashrc
```

#### 4. 驗證 kubectl 設定
```bash
kubectl get pods -A
```

#### 5. 查看節點資訊
觀察目前的運算資源是由哪些節點來運作的：
```bash
$ kubectl get nodes
AME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   12m   v1.33.1
```

可以用以下指令單就某一個 node 看得更細一點：
```bash
$ kubectl describe nodes minikube
Name:               minikube
Roles:              control-plane
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=minikube
                    kubernetes.io/os=linux
                    minikube.k8s.io/commit=f8f52f5de11fc6ad8244afac475e1d0f96841df1-dirty
                    minikube.k8s.io/name=minikube
                    minikube.k8s.io/primary=true
                    minikube.k8s.io/updated_at=2025_06_18T15_29_11_0700
                    minikube.k8s.io/version=v1.36.0
                    node-role.kubernetes.io/control-plane=
                    node.kubernetes.io/exclude-from-external-load-balancers=
Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: unix:///var/run/cri-dockerd.sock
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Wed, 18 Jun 2025 15:29:08 +0000
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  minikube
  AcquireTime:     <unset>
  RenewTime:       Wed, 18 Jun 2025 15:41:45 +0000
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Wed, 18 Jun 2025 15:39:33 +0000   Wed, 18 Jun 2025 15:29:05 +0000   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Wed, 18 Jun 2025 15:39:33 +0000   Wed, 18 Jun 2025 15:29:05 +0000   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Wed, 18 Jun 2025 15:39:33 +0000   Wed, 18 Jun 2025 15:29:05 +0000   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            True    Wed, 18 Jun 2025 15:39:33 +0000   Wed, 18 Jun 2025 15:29:08 +0000   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  192.168.49.2
  Hostname:    minikube
Capacity:
  cpu:                2
  ephemeral-storage:  16764908Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3964656Ki
  pods:               110
Allocatable:
  cpu:                2
  ephemeral-storage:  16764908Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3964656Ki
  pods:               110
System Info:
  Machine ID:                 a427586dfe4f427da40bdebe828a9179
  System UUID:                ecba4891-c4f7-43c6-8d5c-8b57e1d5ff05
  Boot ID:                    45261801-266b-4b62-a2df-021a498b8fdf
  Kernel Version:             4.14.355-277.647.amzn2.x86_64
  OS Image:                   Ubuntu 22.04.5 LTS
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  docker://28.1.1
  Kubelet Version:            v1.33.1
  Kube-Proxy Version:
PodCIDR:                      10.244.0.0/24
PodCIDRs:                     10.244.0.0/24
Non-terminated Pods:          (7 in total)
  Namespace                   Name                                CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                ------------  ----------  ---------------  -------------  ---
  kube-system                 coredns-674b8bbfcf-9jcq5            100m (5%)     0 (0%)      70Mi (1%)        170Mi (4%)     12m
  kube-system                 etcd-minikube                       100m (5%)     0 (0%)      100Mi (2%)       0 (0%)         12m
  kube-system                 kube-apiserver-minikube             250m (12%)    0 (0%)      0 (0%)           0 (0%)         12m
  kube-system                 kube-controller-manager-minikube    200m (10%)    0 (0%)      0 (0%)           0 (0%)         12m
  kube-system                 kube-proxy-6x55d                    0 (0%)        0 (0%)      0 (0%)           0 (0%)         12m
  kube-system                 kube-scheduler-minikube             100m (5%)     0 (0%)      0 (0%)           0 (0%)         12m
  kube-system                 storage-provisioner                 0 (0%)        0 (0%)      0 (0%)           0 (0%)         12m
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                750m (37%)  0 (0%)
  memory             170Mi (4%)  170Mi (4%)
  ephemeral-storage  0 (0%)      0 (0%)
  hugepages-1Gi      0 (0%)      0 (0%)
  hugepages-2Mi      0 (0%)      0 (0%)
Events:
  Type     Reason                   Age   From             Message
  ----     ------                   ----  ----             -------
  Normal   Starting                 12m   kube-proxy
  Normal   Starting                 12m   kubelet          Starting kubelet.
  Warning  CgroupV1                 12m   kubelet          cgroup v1 support is in maintenance mode, please migrate to cgroup v2
  Normal   NodeAllocatableEnforced  12m   kubelet          Updated Node Allocatable limit across pods
  Normal   NodeHasSufficientMemory  12m   kubelet          Node minikube status is now: NodeHasSufficientMemory
  Normal   NodeHasNoDiskPressure    12m   kubelet          Node minikube status is now: NodeHasNoDiskPressure
  Normal   NodeHasSufficientPID     12m   kubelet          Node minikube status is now: NodeHasSufficientPID
  Normal   RegisteredNode           12m   node-controller  Node minikube event: Registered Node minikube in Controller
```
其中，`Non-terminated Pods`的部分會顯示有哪些 pod 被分配到這個 node 之中

---

## 部署應用程式

#### 1. 建立 Deployment
```bash
$ kubectl create deployment hello-minikube --image=k8s.gcr.io/echoserver:1.10
```

#### 2. 檢查 Deployment 狀態
```bash
$ kubectl get deployments
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
hello-minikube   1/1     1            1           20s
```

想要細看的話就用 describe 選項：
```bash
$ kubectl describe deployments hello-minikube
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
hello-minikube   1/1     1            1           20s
[ssm-user@ip-172-31-17-93 bin]$ kubectl describe deployment hello-minikube
Name:                   hello-minikube
Namespace:              default
CreationTimestamp:      Wed, 18 Jun 2025 15:46:38 +0000
Labels:                 app=hello-minikube
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=hello-minikube
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=hello-minikube
  Containers:
   echoserver:
    Image:         k8s.gcr.io/echoserver:1.10
    Port:          <none>
    Host Port:     <none>
    Environment:   <none>
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   hello-minikube-74878c8fcc (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  2m7s  deployment-controller  Scaled up replica set hello-minikube-74878c8fcc from 0 to 1
```

#### 3. 等待 Pod 啟動完成
```bash
# 等待一段時間讓 Pod 完全啟動
$ kubectl get pods
AME                              READY   STATUS    RESTARTS   AGE
hello-minikube-74878c8fcc-b7ttx   1/1     Running   0          3m23s
```

```bash
$ kubectl describe pods [pod_name]
Name:             hello-minikube-74878c8fcc-b7ttx
Namespace:        default
Priority:         0
Service Account:  default
Node:             minikube/192.168.49.2
Start Time:       Wed, 18 Jun 2025 15:46:38 +0000
Labels:           app=hello-minikube
                  pod-template-hash=74878c8fcc
Annotations:      <none>
Status:           Running
IP:               10.244.0.3
IPs:
  IP:           10.244.0.3
Controlled By:  ReplicaSet/hello-minikube-74878c8fcc
Containers:
  echoserver:
    Container ID:   docker://be4fff048121b75d94e963978a52312b051cb403816ce3aa5a01f5ceadb5b44d
    Image:          k8s.gcr.io/echoserver:1.10
    Image ID:       docker-pullable://k8s.gcr.io/echoserver@sha256:cb5c1bddd1b5665e1867a7fa1b5fa843a47ee433bbb75d4293888b71def53229
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Wed, 18 Jun 2025 15:46:44 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-2xp5d (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True
  Initialized                 True
  Ready                       True
  ContainersReady             True
  PodScheduled                True
Volumes:
  kube-api-access-2xp5d:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    Optional:                false
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  4m22s  default-scheduler  Successfully assigned default/hello-minikube-74878c8fcc-b7ttx to minikube
  Normal  Pulling    4m21s  kubelet            Pulling image "k8s.gcr.io/echoserver:1.10"
  Normal  Pulled     4m17s  kubelet            Successfully pulled image "k8s.gcr.io/echoserver:1.10" in 4.212s (4.212s including waiting). Image size: 95361986 bytes.
  Normal  Created    4m16s  kubelet            Created container: echoserver
  Normal  Started    4m16s  kubelet            Started container echoserver
```

#### 4. 查看 Pod 分佈
```bash
$ kubectl get nodes
AME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   23m   v1.33.1
```
還是只有一個 node，表示上面建立的 pod 就在這裡面

用以下指令看詳細一點：
```bash
$ kubectl describe nodes minikube
...
Non-terminated Pods:          (8 in total)
  Namespace                   Name                                CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                ------------  ----------  ---------------  -------------  ---
  default                     hello-minikube-74878c8fcc-b7ttx     0 (0%)        0 (0%)      0 (0%)           0 (0%)         7m1s
  kube-system                 coredns-674b8bbfcf-9jcq5            100m (5%)     0 (0%)      70Mi (1%)        170Mi (4%)     24m
  kube-system                 etcd-minikube                       100m (5%)     0 (0%)      100Mi (2%)       0 (0%)         24m
  kube-system                 kube-apiserver-minikube             250m (12%)    0 (0%)      0 (0%)           0 (0%)         24m
  kube-system                 kube-controller-manager-minikube    200m (10%)    0 (0%)      0 (0%)           0 (0%)         24m
  kube-system                 kube-proxy-6x55d                    0 (0%)        0 (0%)      0 (0%)           0 (0%)         24m
  kube-system                 kube-scheduler-minikube             100m (5%)     0 (0%)      0 (0%)           0 (0%)         24m
  kube-system                 storage-provisioner                 0 (0%)        0 (0%)      0 (0%)           0 (0%)         24m
...
```
可以看到 pod `hello-minikube` 有在清單上，透過這樣的方式，就可以知道部署的 pod 在哪個 node 上了

---

## 服務管理

#### 1. 建立 Service
```bash
$ kubectl expose deployment hello-minikube --port=8080
service/hello-minikube exposed
```

#### 2. 檢查 Service 狀態
```bash
$ kubectl get services
NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
hello-minikube   ClusterIP   10.100.51.36   <none>        8080/TCP   10s
kubernetes       ClusterIP   10.96.0.1      <none>        443/TCP    27m
```

詳細資訊：
```bash
$ kubectl describe services hello-minikube
Name:                     hello-minikube
Namespace:                default
Labels:                   app=hello-minikube
Annotations:              <none>
Selector:                 app=hello-minikube
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.100.51.36
IPs:                      10.100.51.36
Port:                     <unset>  8080/TCP
TargetPort:               8080/TCP
Endpoints:                10.244.0.3:8080
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>
```

`ClusterIP` type 暗示了這個 cluster 不會將 cluster-ip 對外開放給外部使用，譬如此時如果執行:
```bash
$ curl 10.100.51.36
```
是不會有回應的

---

## 網路連線測試

#### 1. 直接測試 Container 連線
```bash
# 查看 Docker 容器
$ docker ps
d86c55dddc18   gcr.io/k8s-minikube/kicbase:v0.0.47   "/usr/local/bin/entr…"   32 minutes ago   Up 32 minutes   127.0.0.1:32777->22/tcp, 127.0.0.1:32776->2376/tcp, 127.0.0.1:32775->5000/tcp, 127.0.0.1:32774->8443/tcp, 127.0.0.1:32773->32443/tcp   minikube
```
透過 minikube 創造的 cluster 是在 minikube container 之中，所以對於上面的 ClusterIP type 來說，就要在 cluster 當中才可以連得到

```bash
# 進入 MiniKube 容器測試
$ docker exec -it minikube bash
root@minikube:/$ curl 10.100.51.36:8080


Hostname: hello-minikube-74878c8fcc-b7ttx

Pod Information:
        -no pod information available-

Server values:
        server_version=nginx: 1.13.3 - lua: 10008

Request Information:
        client_address=10.244.0.1
        method=GET
        real path=/
        query=
        request_version=1.1
        request_scheme=http
        request_uri=http://10.100.51.36:8080/

Request Headers:
        accept=*/*
        host=10.100.51.36:8080
        user-agent=curl/7.81.0

Request Body:
        -no body in request-
```

#### 2. 設定 Port Forwarding
```bash
# 建立 local 到 cluster 的連線, & 代表在背景執行
$ kubectl port-forward service/hello-minikube 8081:8080 &
[1] 43258
[ssm-user@ip-172-31-17-93 bin]$ Forwarding from 127.0.0.1:8081 -> 8080
Forwarding from [::1]:8081 -> 8080
```

#### 3. 測試本地連線
```bash
$ curl localhost:8081
Handling connection for 8081


Hostname: hello-minikube-74878c8fcc-b7ttx

Pod Information:
        -no pod information available-

Server values:
        server_version=nginx: 1.13.3 - lua: 10008

Request Information:
        client_address=127.0.0.1
        method=GET
        real path=/
        query=
        request_version=1.1
        request_scheme=http
        request_uri=http://localhost:8080/

Request Headers:
        accept=*/*
        host=localhost:8081
        user-agent=curl/8.3.0

Request Body:
        -no body in request-
```

#### 4. 關閉 Port Forwarding
```bash
# 查看背景程序
$ ps
    PID TTY          TIME CMD
  30341 pts/1    00:00:00 sh
  43258 pts/1    00:00:00 minikube
  43264 pts/1    00:00:00 kubectl
  43602 pts/1    00:00:00 ps

# 終止 kubectl port-forward 程序
$ kill 43258
[1]+  Terminated              minikube kubectl -- port-forward service/hello-minikube 8081:8080

# 確認程序已終止
$ ps
    PID TTY          TIME CMD
  30341 pts/1    00:00:00 sh
  43264 pts/1    00:00:00 kubectl
  43804 pts/1    00:00:00 ps
```

---

## 清理資源

#### 1. 刪除 Service
```bash
$ kubectl get services
NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
hello-minikube   ClusterIP   10.100.51.36   <none>        8080/TCP   14m
kubernetes       ClusterIP   10.96.0.1      <none>        443/TCP    41m

$ kubectl delete service hello-minikube
service "hello-minikube" deleted

$ kubectl get service
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   41m
```

#### 2. 刪除 Deployment
```bash
$ kubectl get deployments
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
hello-minikube   1/1     1            1           24m

$ kubectl delete deployment hello-minikube
deployment.apps "hello-minikube" deleted

$ kubectl get deployments
No resources found in default namespace.

$ kubectl get pods
NAME                              READY   STATUS        RESTARTS   AGE
hello-minikube-74878c8fcc-b7ttx   1/1     Terminating   0          24m

# 再等一會
$ kubectl get pods
No resources found in default namespace.
```

#### 3. 停止並刪除 Cluster (選擇性)
```bash
$ minikube status
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured

$ minikube stop
Stopping node "minikube"  ...
Powering off "minikube" via SSH ...
error: lost connection to pod
1 node stopped.

$ minikube delete
Deleting "minikube" in docker ...
Deleting container "minikube" ...
Removing /home/ssm-user/.minikube/machines/minikube ...
Removed all traces of the "minikube" cluster.

$ minikube status
Profile "minikube" not found. Run "minikube profile list" to view all profiles.
To start a cluster, run: "minikube start"
```
---

## 常用指令速查

#### Cluster 管理
- `minikube start` - 啟動 cluster
- `minikube stop` - 停止 cluster
- `minikube delete` - 刪除 cluster
- `minikube status` - 查看 cluster 狀態

#### Pod 管理
- `kubectl get pods` - 列出所有 pods
- `kubectl describe pod [pod_name]` - 查看 pod 詳細資訊
- `kubectl logs [pod_name]` - 查看 pod 日誌

#### Service 管理
- `kubectl get services` - 列出所有 services
- `kubectl expose deployment [name] --port=[port]` - 建立 service
- `kubectl delete service [name]` - 刪除 service

#### 網路連線
- `kubectl port-forward service/[service_name] [local_port]:[service_port]` - 建立 port forwarding

---

## 注意事項

1. **權限設定**: 確保使用者已加入 docker 群組
2. **網路連線**: 需要穩定的網路連線下載 Docker 映像檔
3. **資源需求**: MiniKube 需要足夠的記憶體和 CPU 資源
4. **防火牆設定**: 確保相關埠號未被防火牆阻擋

---
