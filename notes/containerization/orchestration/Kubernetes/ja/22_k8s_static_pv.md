# Kubernetes Static Persistent Volume 実装

[English](../en/22_k8s_static_pv.md) | [繁體中文](../zh-tw/22_k8s_static_pv.md) | [日本語](../ja/22_k8s_static_pv.md) | [インデックスに戻る](../README.md)

## 概要
このガイドでは、KubernetesでStatic Persistent Volume (PV)とPersistent Volume Claim (PVC)を使用してデータの永続化を実現する方法を説明します。

## 前提条件

#### Docker環境の起動
```bash
# dockerサービスの開始
$ sudo service docker start
$ docker ps

# minikubeクラスターの作成
$ minikube start --driver docker
$ minikube status
```

## Persistent Volume (PV) の作成

#### PV定義ファイルの作成
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

#### PVのデプロイ
```bash
$ kubectl apply -f simple-volume-pv.yaml

$ kubectl get pv
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
app-pv   2Gi        RWX            Retain           Available           sc-001         <unset>                          12s
```

## Persistent Volume Claim (PVC) の作成

#### PVC定義ファイルの作成
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

#### PVCのデプロイ
```bash
$ kubectl apply -f simple-volume-pvc.yaml

$ kubectl get pvc
NAME      STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
app-pvc   Bound    app-pv   2Gi        RWX            sc-001         <unset>                 4s
```

## PVCを使用するDeploymentの作成

#### Deployment定義ファイルの作成
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

#### アプリケーションのデプロイ
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
# マウントパス設定の確認
```

## Volumeライフサイクルのテスト

#### テストファイルの作成
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

#### データ永続性のテスト
```bash
# 全てのPodを削除
$ kubectl delete pods --all
pod "app-deployment-58449898c5-9tvgb" deleted
pod "app-deployment-58449898c5-9zs8p" deleted
pod "app-deployment-58449898c5-jfr9k" deleted

$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-58449898c5-dklg5   1/1     Running   0          37s
app-deployment-58449898c5-hcjhq   1/1     Running   0          37s
app-deployment-58449898c5-vxvwd   1/1     Running   0          37s

# 新しいPodでファイルが存在するかチェック
$ kubectl exec -it [pod_name] -- ls /app/data
file001.txt
```

## 実際のストレージ場所の確認

#### PV詳細情報の確認
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
    Path:          /data  # これはminikube内のパスを指します
    HostPathType:  DirectoryOrCreate
Events:            <none>
# ソースパスの確認
```

#### minikube内の実際のファイルの確認
```bash
$ docker ps
$ docker exec -it minikube ls /data
file001.txt
```

## リソースのクリーンアップ

#### 全リソースのクリーンアップ
```bash
$ kubectl delete deployments --all
$ kubectl delete pvc --all
$ kubectl delete pv --all
```

## 重要概念の説明

#### Static PV vs Dynamic PV
- **Static PV**: 管理者が手動でPVを作成し、PVCがこれらの事前に作成されたPVにバインドできます
- **Dynamic PV**: StorageClassを使用してPVを自動的に作成します

#### 主要コンポーネント
- **PersistentVolume (PV)**: 実際のストレージリソースを定義します
- **PersistentVolumeClaim (PVC)**: アプリケーションのストレージリソースへのリクエストです
- **StorageClass**: ストレージタイプとプロビジョナーを定義します

#### アクセスモード
- **ReadWriteOnce (RWO)**: 単一ノードでの読み書き
- **ReadOnlyMany (ROX)**: 複数ノードでの読み取り専用
- **ReadWriteMany (RWM)**: 複数ノードでの読み書き