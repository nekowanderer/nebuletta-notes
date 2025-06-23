# Kubernetes Zero-downtime Rollback 實作

[English](../en/20_k8s_zero_downtime_rollback.md) | [繁體中文](../zh-tw/20_k8s_zero_downtime_rollback.md) | [日本語](../ja/20_k8s_zero_downtime_rollback.md) | [回到索引](../README.md)

## 前置準備

接續 [19_k8s_rolling_updates](./19_k8s_rolling_updates.md)

## Rollback 操作流程

### 1. 監控 ReplicaSet 狀態
```bash
$ kubectl get rs
NAME                        DESIRED   CURRENT   READY   AGE
app-deployment-6f975d88d9   3         3         3       63s   # v2
app-deployment-7f57cfb4b6   0         0         0       119s  # v1
```

### 2. 查看 Deployment 歷史記錄
```bash
$ kubectl get deployments
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
app-deployment   3/3     3            3           7m17s

$ kubectl rollout history deployment/app-deployment
deployment.apps/app-deployment
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

### 3. 執行 Rollback 操作
```bash
$ kubectl rollout undo deployment/app-deployment
deployment.apps/app-deployment rolled back
```

### 4. 驗證 Rollback 結果

#### 檢查 Pod 狀態
```bash
$ kubectl get rs
NAME                        DESIRED   CURRENT   READY   AGE
app-deployment-6f975d88d9   3         3         3       7m32s
app-deployment-7f57cfb4b6   1         1         0       8m28s
app-deployment-7f57cfb4b6   1         1         1       8m36s
app-deployment-6f975d88d9   2         3         3       7m40s
app-deployment-7f57cfb4b6   2         1         1       8m36s
app-deployment-6f975d88d9   2         3         3       7m40s
app-deployment-7f57cfb4b6   2         1         1       8m36s
app-deployment-6f975d88d9   2         2         2       7m40s
app-deployment-7f57cfb4b6   2         2         1       8m36s
app-deployment-7f57cfb4b6   2         2         2       8m52s
app-deployment-6f975d88d9   1         2         2       7m56s
app-deployment-6f975d88d9   1         2         2       7m56s
app-deployment-6f975d88d9   1         1         1       7m56s
app-deployment-7f57cfb4b6   3         2         2       8m52s
app-deployment-7f57cfb4b6   3         2         2       8m52s
app-deployment-7f57cfb4b6   3         3         2       8m52s
app-deployment-7f57cfb4b6   3         3         3       9m7s
app-deployment-6f975d88d9   0         1         1       8m11s
app-deployment-6f975d88d9   0         1         1       8m11s
app-deployment-6f975d88d9   0         0         0       8m11s

$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-6f975d88d9-b4p4g   1/1     Running   0          6m51s
app-deployment-6f975d88d9-dql9k   1/1     Running   0          7m21s
app-deployment-6f975d88d9-ghjmh   1/1     Running   0          7m6s
app-deployment-7f57cfb4b6-2kf7t   0/1     Pending   0          0s
app-deployment-7f57cfb4b6-2kf7t   0/1     Pending   0          0s
app-deployment-7f57cfb4b6-2kf7t   0/1     ContainerCreating   0          0s
app-deployment-7f57cfb4b6-2kf7t   0/1     Running             0          2s
app-deployment-7f57cfb4b6-2kf7t   1/1     Running             0          14s
app-deployment-6f975d88d9-dql9k   1/1     Terminating         0          7m40s
app-deployment-7f57cfb4b6-gnkbx   0/1     Pending             0          0s
app-deployment-7f57cfb4b6-gnkbx   0/1     Pending             0          0s
app-deployment-7f57cfb4b6-gnkbx   0/1     ContainerCreating   0          0s
app-deployment-7f57cfb4b6-gnkbx   0/1     Running             0          11s
app-deployment-7f57cfb4b6-gnkbx   1/1     Running             0          16s
app-deployment-6f975d88d9-b4p4g   1/1     Terminating         0          7m26s
app-deployment-7f57cfb4b6-wbqfh   0/1     Pending             0          0s
app-deployment-7f57cfb4b6-wbqfh   0/1     Pending             0          0s
app-deployment-7f57cfb4b6-wbqfh   0/1     ContainerCreating   0          0s
app-deployment-7f57cfb4b6-wbqfh   0/1     Running             0          3s
app-deployment-7f57cfb4b6-wbqfh   0/1     Running             0          10s
app-deployment-6f975d88d9-dql9k   0/1     Error               0          8m11s
app-deployment-7f57cfb4b6-wbqfh   1/1     Running             0          15s
app-deployment-6f975d88d9-ghjmh   1/1     Terminating         0          7m56s
app-deployment-6f975d88d9-dql9k   0/1     Error               0          8m11s
app-deployment-6f975d88d9-dql9k   0/1     Error               0          8m11s
app-deployment-6f975d88d9-b4p4g   0/1     Error               0          7m57s
app-deployment-6f975d88d9-ghjmh   0/1     Error               0          8m27s
```

#### 確認 replica set 狀態與內容
```base
$ kubectl get rs
NAME                        DESIRED   CURRENT   READY   AGE
app-deployment-6f975d88d9   0         0         0       10m
app-deployment-7f57cfb4b6   3         3         3       10m


$ kubectl describe rs app-deployment-7f57cfb4b6
Name:           app-deployment-7f57cfb4b6
Namespace:      default
Selector:       app=app-pod,pod-template-hash=7f57cfb4b6
Labels:         app=app-pod
                pod-template-hash=7f57cfb4b6
Annotations:    deployment.kubernetes.io/desired-replicas: 3
                deployment.kubernetes.io/max-replicas: 4
                deployment.kubernetes.io/revision: 3
                deployment.kubernetes.io/revision-history: 1
Controlled By:  Deployment/app-deployment
Replicas:       3 current / 3 desired
Pods Status:    3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=app-pod
           pod-template-hash=7f57cfb4b6
  Containers:
   app-container:
    Image:         uopsdod/k8s-hostname-amd64-beta:v1
    Port:          80/TCP
    Host Port:     0/TCP
    Liveness:      http-get http://:80/healthcheck delay=0s timeout=1s period=1s #success=1 #failure=5
    Readiness:     http-get http://:80/healthcheck_dependency delay=0s timeout=1s period=1s #success=5 #failure=7
    Startup:       http-get http://:80/healthcheck delay=0s timeout=1s period=10s #success=1 #failure=30
    Environment:   <none>
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Events:
  Type    Reason            Age    From                   Message
  ----    ------            ----   ----                   -------
  Normal  SuccessfulCreate  11m    replicaset-controller  Created pod: app-deployment-7f57cfb4b6-9vb4p
  Normal  SuccessfulCreate  11m    replicaset-controller  Created pod: app-deployment-7f57cfb4b6-7qt8n
  Normal  SuccessfulCreate  11m    replicaset-controller  Created pod: app-deployment-7f57cfb4b6-vngmf
  Normal  SuccessfulDelete  10m    replicaset-controller  Deleted pod: app-deployment-7f57cfb4b6-vngmf
  Normal  SuccessfulDelete  9m59s  replicaset-controller  Deleted pod: app-deployment-7f57cfb4b6-9vb4p
  Normal  SuccessfulDelete  9m44s  replicaset-controller  Deleted pod: app-deployment-7f57cfb4b6-7qt8n
  Normal  SuccessfulCreate  3m3s   replicaset-controller  Created pod: app-deployment-7f57cfb4b6-2kf7t
  Normal  SuccessfulCreate  2m49s  replicaset-controller  Created pod: app-deployment-7f57cfb4b6-gnkbx
  Normal  SuccessfulCreate  2m33s  replicaset-controller  Created pod: app-deployment-7f57cfb4b6-wbqfh
```

#### 確認 Container Image 版本
```bash
$ kubectl describe pod [pod_id] | grep Image:
    Image:          uopsdod/k8s-hostname-amd64-beta:v1
```

## 清理資源

#### 刪除所有 Deployments
```bash
$ kubectl delete deployments --all
```

## 注意事項

- Rollback 操作會自動將 Deployment 回滾到上一個版本
- 確保在執行 Rollback 前已備份重要資料
- 建議在測試環境中先驗證 Rollback 流程
- 但是，最好的做法還是直接把 yaml 檔案改成要退回的版本然後重新部署，因為這樣才可以維持版本控制的一致性
