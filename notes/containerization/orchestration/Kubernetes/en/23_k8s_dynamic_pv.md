# Kubernetes Dynamic Persistent Volume Implementation

[English](../en/23_k8s_dynamic_pv.md) | [繁體中文](../zh-tw/23_k8s_dynamic_pv.md) | [日本語](../ja/23_k8s_dynamic_pv.md) | [Back to Index](../README.md)

This note explains how to implement dynamic Persistent Volume (PV) provisioning using `StorageClass` and `PersistentVolumeClaim` (PVC). Unlike static PVs, dynamic PVs do not require an administrator to manually create PVs in advance. Instead, the `StorageClass` automatically provisions a PV based on the PVC request.

### Prerequisites

```bash
$ sudo service docker start
$ docker ps

$ minikube start --driver docker
$ minikube status
```

### Verify the Default StorageClass

Kubernetes uses `StorageClass` to define different types of storage. When Minikube is initialized, it automatically creates a default `StorageClass` named `standard`.

```bash
# View all available StorageClasses in the current cluster
$ kubectl get storageclass
NAME                 PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
standard (default)   k8s.io/minikube-hostpath   Delete          Immediate           false                  5d9h
```

**Observations and Inferences:**
- The default `standard` StorageClass is the foundation for dynamic PVs. When a PVC is created and specifies this `storageClassName`, the system will automatically provision a corresponding PV for it.

### Create and Deploy a Persistent Volume Claim (PVC)

Create a file named `dynamic-volume-pvc.yaml` and define the PVC specifications within it.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc
spec:
  storageClassName: standard # Specify the use of the standard StorageClass
  accessModes:
    - ReadWriteMany # Allows multiple nodes to read and write simultaneously
  resources:
    requests:
      storage: 2Gi
```

##### Deploy the PVC:
Use the `kubectl apply` command to create this PVC resource.

```bash
$ kubectl apply -f dynamic-volume-pvc.yaml

$ kubectl get pvc
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
app-pvc   Bound    pvc-4600ab64-eee0-49ce-8257-b5ea2163684e   2Gi        RWX            standard       <unset>                 6s
```

### Verify the Dynamically Provisioned Persistent Volume (PV)

Once the PVC is successfully deployed, the `StorageClass` will automatically create a corresponding PV for us.

```bash
# View the PV automatically created by the StorageClass
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pvc-4600ab64-eee0-49ce-8257-b5ea2163684e   2Gi        RWX            Delete           Bound    default/app-pvc   standard       <unset>                          30s
```

**Observations and Inferences:**
- A new PV is automatically created with the status `Bound`, indicating it has successfully been bound to the `app-pvc` PVC.
- This demonstrates the core mechanism of dynamic PVs: **PVC -> StorageClass -> Automatic PV Provisioning**.

### Deploy an Application and Mount the Volume

Now, we can deploy an application (e.g., a Deployment) and mount the previously created PVC into the Pod, allowing the application to use the storage space.

```bash
# Deploy the application and mount the PVC
$ kubectl apply -f simple-deployment-volume.yaml

# Check the Deployment status
$ kubectl get deployments
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
app-deployment   3/3     3            3           29s

$ kubectl describe deployments app-deployment | grep Mounts: -A 7
    Mounts:
      /app/data from app-volume (rw)
  Volumes:
   app-volume:
    Type:          PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:     app-pvc
    ReadOnly:      false
  Node-Selectors:  <none>
```

**Note:**
- In `simple-deployment-volume.yaml`, ensure that the `volumes` and `volumeMounts` are configured correctly to mount `app-pvc` to the specified path within the container (e.g., `/app/data`).

### Test Volume Lifecycle and Data Persistence

To verify that the data is truly persistent, we can perform the following tests:

##### Write Data in a Pod:
Exec into one of the Pods and create a file in the mounted directory.

```bash
# Get Pod names
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-58449898c5-g4499   1/1     Running   0          2m57s
app-deployment-58449898c5-jxzsb   1/1     Running   0          2m57s
app-deployment-58449898c5-tb2mm   1/1     Running   0          2m57s

# Exec into a Pod and create a test file
$ kubectl exec -it [pod_name] -- touch /app/data/file002.txt
$ kubectl exec -it [pod_name] -- ls /app/data
```
At this point, `file002.txt` should be successfully created.

##### Delete and Recreate Pods:
Delete all Pods and let the Deployment automatically recreate new ones.

```bash
$ kubectl delete pods --all
pod "app-deployment-58449898c5-g4499" deleted
pod "app-deployment-58449898c5-jxzsb" deleted
pod "app-deployment-58449898c5-tb2mm" deleted

$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-58449898c5-47dqj   1/1     Running   0          46s
app-deployment-58449898c5-fhcvb   1/1     Running   0          46s
app-deployment-58449898c5-hd46z   1/1     Running   0          46s
```

##### Verify Data Persistence:
Exec into a newly created Pod and check if the previously written file exists.

```bash
# Exec into a new Pod
$ kubectl exec -it [new_pod_name] -- ls /app/data
file002.txt
```
We will find that `file002.txt` still exists, proving that even if the Pod is destroyed and recreated, the data persists in the storage provided by the PV.

### Resource Cleanup

After testing, remember to delete all created resources to free up space.

```bash
$ kubectl delete deployments --all
$ kubectl delete pvc --all
$ kubectl delete pv --all
```