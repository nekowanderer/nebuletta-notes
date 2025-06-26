# Kubernetes Dynamic Persistent Volume 實作

[English](../en/23_k8s_dynamic_pv.md) | [繁體中文](../zh-tw/23_k8s_dynamic_pv.md) | [日本語](../ja/23_k8s_dynamic_pv.md) | [回到索引](../README.md)

這份筆記將說明如何透過 `StorageClass` 與 `PersistentVolumeClaim` (PVC) 來實現動態的 Persistent Volume (PV) 生成。與靜態 PV 不同，動態 PV 不需要管理者預先手動建立 PV，而是由 `StorageClass` 根據 PVC 的請求自動配置。

### 前置準備

```bash
$ sudo service docker start
$ docker ps

$ minikube start --driver docker
$ minikube status
```

### 確認預設的 StorageClass

Kubernetes 使用 `StorageClass` 來定義不同類型的儲存。Minikube 在初始化時會自動建立一個名為 `standard` 的預設 `StorageClass`。

```bash
# 查看當前 cluster 中所有可用的 StorageClass
$ kubectl get storageclass
NAME                 PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
standard (default)   k8s.io/minikube-hostpath   Delete          Immediate           false                  5d9h
```

觀察與推論：
- 預設的 `standard` StorageClass 是動態 PV 的基礎。當一個 PVC 被建立且指定了這個 `storageClassName`，系統將會自動為其配置一個對應的 PV。

### 建立並部署 Persistent Volume Claim (PVC)

建立一個名為 `dynamic-volume-pvc.yaml` 的檔案，並在其中定義 PVC 的規格。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc
spec:
  storageClassName: standard # 指定使用 standard StorageClass
  accessModes:
    - ReadWriteMany # 允許多個節點同時讀寫
  resources:
    requests:
      storage: 2Gi
```

##### 部署 PVC：
使用 `kubectl apply` 指令來建立這個 PVC 資源。

```bash
$ kubectl apply -f dynamic-volume-pvc.yaml

$ kubectl get pvc
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
app-pvc   Bound    pvc-4600ab64-eee0-49ce-8257-b5ea2163684e   2Gi        RWX            standard       <unset>                 6s
```

### 驗證動態生成的 Persistent Volume (PV)

當 PVC 成功部署後，`StorageClass` 會自動為我們建立一個對應的 PV。

```bash
# 查看由 StorageClass 自動建立的 PV
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pvc-4600ab64-eee0-49ce-8257-b5ea2163684e   2Gi        RWX            Delete           Bound    default/app-pvc   standard       <unset>                          30s
```

觀察與推論：
- 我們會看到一個新的 PV 被自動建立，其狀態為 `Bound`，表示已經成功與 `app-pvc` 這個 PVC 綁定。
- 這證明了動態 PV 的核心機制：**PVC -> StorageClass -> 自動生成 PV**。

### 部署應用程式並掛載 Volume

現在，我們可以部署一個應用程式（例如 Deployment），並將先前建立的 PVC 掛載到 Pod 中，讓應用程式可以使用該儲存空間。

```bash
# 部署應用程式並將 PVC 掛載進去
$ kubectl apply -f simple-deployment-volume.yaml

# 確認 Deployment 狀態
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

注意事項：
- 在 `simple-deployment-volume.yaml` 中，需要確保 `volumes` 和 `volumeMounts` 的設定是正確的，將 `app-pvc` 掛載到容器內的指定路徑（例如 `/app/data`）。

### 測試 Volume 的生命週期與資料持久性

為了驗證資料是否真的被持久化儲存，我們可以執行以下測試：

##### 在 Pod 中寫入資料：
進入其中一個 Pod，並在掛載的目錄中建立一個檔案。

```bash
# 取得 Pod 名稱
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-58449898c5-g4499   1/1     Running   0          2m57s
app-deployment-58449898c5-jxzsb   1/1     Running   0          2m57s
app-deployment-58449898c5-tb2mm   1/1     Running   0          2m57s

# 進入 Pod 並建立測試檔案
$ kubectl exec -it [pod_name] -- touch /app/data/file002.txt
$ kubectl exec -it [pod_name] -- ls /app/data
```
此時應可看到 `file002.txt` 已成功建立。

##### 刪除並重建 Pod：
刪除所有 Pod，讓 Deployment 自動重建新的 Pod。

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

##### 驗證資料是否存在：
進入新建立的 Pod，檢查先前寫入的檔案是否存在。

```bash
# 進入新的 Pod
$ kubectl exec -it [new_pod_name] -- ls /app/data
file002.txt
```
我們會發現 `file002.txt` 依然存在，這證明了即使 Pod 被銷毀重建，資料仍然被保存在由 PV 提供支援的儲存空間中。

### 資源清理

測試完成後，記得刪除所有建立的資源，以釋放空間。

```bash
$ kubectl delete deployments --all
$ kubectl delete pvc --all
$ kubectl delete pv --all
```
