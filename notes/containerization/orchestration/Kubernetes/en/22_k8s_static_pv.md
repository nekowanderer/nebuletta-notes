# Kubernetes Static Persistent Volume Implementation

[English](../en/22_k8s_static_pv.md) | [繁體中文](../zh-tw/22_k8s_static_pv.md) | [日本語](../ja/22_k8s_static_pv.md) | [Back to Index](../README.md)


## Overview
This guide demonstrates how to use Static Persistent Volume (PV) and Persistent Volume Claim (PVC) in Kubernetes to achieve data persistence.

## Prerequisites

#### Start Docker Environment
```bash
# Start docker service
$ sudo service docker start
$ docker ps

# Create minikube cluster
$ minikube start --driver docker
$ minikube status
```

## Create Persistent Volume (PV)

#### Create PV Definition File
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

#### Deploy PV
```bash
$ kubectl apply -f simple-volume-pv.yaml

$ kubectl get pv
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
app-pv   2Gi        RWX            Retain           Available           sc-001         <unset>                          12s
```

## Create Persistent Volume Claim (PVC)

#### Create PVC Definition File
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

#### Deploy PVC
```bash
$ kubectl apply -f simple-volume-pvc.yaml

$ kubectl get pvc
NAME      STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
app-pvc   Bound    app-pv   2Gi        RWX            sc-001         <unset>                 4s
```

## Create Deployment Using PVC

#### Create Deployment Definition File
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

#### Deploy Application
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
# Check mount path configuration
```

## Test Volume Lifecycle

#### Create Test File
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

#### Test Data Persistence
```bash
# Delete all pods
$ kubectl delete pods --all
pod "app-deployment-58449898c5-9tvgb" deleted
pod "app-deployment-58449898c5-9zs8p" deleted
pod "app-deployment-58449898c5-jfr9k" deleted

$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-58449898c5-dklg5   1/1     Running   0          37s
app-deployment-58449898c5-hcjhq   1/1     Running   0          37s
app-deployment-58449898c5-vxvwd   1/1     Running   0          37s

# Check if the file still exists in new pods
$ kubectl exec -it [pod_name] -- ls /app/data
file001.txt
```

## Verify Actual Storage Location

#### Check PV Details
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
    Path:          /data  # This refers to the path inside minikube
    HostPathType:  DirectoryOrCreate
Events:            <none>
# Check source path
```

#### Check Actual Files in Minikube
```bash
$ docker ps
$ docker exec -it minikube ls /data
file001.txt
```

## Resource Cleanup

#### Clean Up All Resources
```bash
$ kubectl delete deployments --all
$ kubectl delete pvc --all
$ kubectl delete pv --all
```

## Important Concept Explanations

#### Static PV vs Dynamic PV
- **Static PV**: Administrators manually create PVs, then PVCs can bind to these pre-created PVs
- **Dynamic PV**: Uses StorageClass to automatically create PVs

#### Key Components
- **PersistentVolume (PV)**: Defines the actual storage resource
- **PersistentVolumeClaim (PVC)**: Application's request for storage resources
- **StorageClass**: Defines storage type and provisioner

#### Access Modes
- **ReadWriteOnce (RWO)**: Single node read-write
- **ReadOnlyMany (ROX)**: Multi-node read-only
- **ReadWriteMany (RWM)**: Multi-node read-write