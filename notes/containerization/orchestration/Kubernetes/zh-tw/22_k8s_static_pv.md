# Kubernetes Static Persistent Volume 實作

[English](../en/22_k8s_static_pv.md) | [繁體中文](../zh-tw/22_k8s_static_pv.md) | [日本語](../ja/22_k8s_static_pv.md) | [回到索引](../README.md)


## 概述
本篇將展示如何在 Kubernetes 中使用 Static Persistent Volume (PV) 和 Persistent Volume Claim (PVC) 來實現資料持久化。

## 前置準備

#### 啟動 Docker 環境
```bash
# 啟動 docker 服務
$ sudo service docker start
$ docker ps

# 建立 minikube cluster
$ minikube start --driver docker
$ minikube status
```

## 建立 Persistent Volume (PV)

#### 建立 PV 定義檔
```bash
$ vi simple-volume-pv.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: app-pv
spec:
  storageClassName: sc-001
  volumeMode: Filesystem
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: /data
    type: DirectoryOrCreate
```

#### 部署 PV
```bash
$ kubectl apply -f simple-volume-pv.yaml

$ kubectl get pv
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
app-pv   2Gi        RWX            Retain           Available           sc-001         <unset>                          12s
```

## 建立 Persistent Volume Claim (PVC)

#### 建立 PVC 定義檔
```bash
$ vi simple-volume-pvc.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc
spec:
  storageClassName: sc-001
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
```

#### 部署 PVC
```bash
$ kubectl apply -f simple-volume-pvc.yaml

$ kubectl get pvc
NAME      STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
app-pvc   Bound    app-pv   2Gi        RWX            sc-001         <unset>                 4s
```

## 建立使用 PVC 的 Deployment

#### 建立 Deployment 定義檔
```bash
$ cp simple-deployment.yaml simple-deployment-volume.yaml
$ vi simple-deployment-volume.yaml
```

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
        volumeMounts:
          - name: app-volume
            mountPath: /app/data
      volumes:
        - name: app-volume
          persistentVolumeClaim:
            claimName: app-pvc
```

#### 部署應用程式
```bash
$ kubectl apply -f simple-deployment-volume.yaml

$ kubectl get deployments
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
app-deployment   3/3     3            3           4s

$ kubectl describe deployments app-deployment | grep -A 5
    Mounts:
      /app/data from app-volume (rw)
  Volumes:
   app-volume:
    Type:          PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:     app-pvc
# 檢查掛載路徑設定
```

## 測試 Volume 生命週期

#### 建立測試檔案
```bash
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-58449898c5-9tvgb   1/1     Running   0          2m4s
app-deployment-58449898c5-9zs8p   1/1     Running   0          2m4s
app-deployment-58449898c5-jfr9k   1/1     Running   0          2m4s

$ kubectl exec -it [pod_name] -- touch /app/data/file001.txt

$ kubectl exec -it [pod_name] -- ls /app/data
file001.txt
```

#### 測試資料持久性
```bash
# 刪除所有 pods
$ kubectl delete pods --all
pod "app-deployment-58449898c5-9tvgb" deleted
pod "app-deployment-58449898c5-9zs8p" deleted
pod "app-deployment-58449898c5-jfr9k" deleted

$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-58449898c5-dklg5   1/1     Running   0          37s
app-deployment-58449898c5-hcjhq   1/1     Running   0          37s
app-deployment-58449898c5-vxvwd   1/1     Running   0          37s

# 檢查新 pod 中是否仍有檔案
$ kubectl exec -it [pod_name] -- ls /app/data
file001.txt
```

## 驗證實際儲存位置

#### 檢查 PV 詳細資訊
```bash
$ kubectl describe pv
Name:            app-pv
Labels:          <none>
Annotations:     pv.kubernetes.io/bound-by-controller: yes
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:    sc-001
Status:          Bound
Claim:           default/app-pvc
Reclaim Policy:  Retain
Access Modes:    RWX
VolumeMode:      Filesystem
Capacity:        2Gi
Node Affinity:   <none>
Message:
Source:
    Type:          HostPath (bare host directory volume)
    Path:          /data  # 這個是指 minikube 裡面的路徑
    HostPathType:  DirectoryOrCreate
Events:            <none>
# 檢查來源路徑
```

#### 檢查 minikube 中的實際檔案
```bash
$ docker ps
$ docker exec -it minikube ls /data
file001.txt
```

## 資源清理

#### 清理所有資源
```bash
$ kubectl delete deployments --all
$ kubectl delete pvc --all
$ kubectl delete pv --all
```

## 重要概念說明

#### Static PV vs Dynamic PV
- **Static PV**: 管理員手動建立 PV，然後 PVC 可以綁定到這些預先建立的 PV
- **Dynamic PV**: 使用 StorageClass 自動建立 PV

#### 關鍵元件
- **PersistentVolume (PV)**: 定義實際的儲存資源
- **PersistentVolumeClaim (PVC)**: 應用程式對儲存資源的請求
- **StorageClass**: 定義儲存類型和供應商

#### 存取模式 (Access Modes)
- **ReadWriteOnce (RWO)**: 單一節點可讀寫
- **ReadOnlyMany (ROX)**: 多節點唯讀
- **ReadWriteMany (RWM)**: 多節點可讀寫
