# Kubernetes Dynamic Persistent Volume 実装

[English](../en/23_k8s_dynamic_pv.md) | [繁體中文](../zh-tw/23_k8s_dynamic_pv.md) | [日本語](../ja/23_k8s_dynamic_pv.md) | [インデックスに戻る](../README.md)

このノートでは、`StorageClass` と `PersistentVolumeClaim` (PVC) を使用して、動的な Persistent Volume (PV) のプロビジョニングを実装する方法について説明します。静的 PV とは異なり、動的 PV では管理者が事前に手動で PV を作成する必要がありません。代わりに、`StorageClass` が PVC のリクエストに基づいて PV を自動的にプロビジョニングします。

### 前提条件

```bash
$ sudo service docker start
$ docker ps

$ minikube start --driver docker
$ minikube status
```

### デフォルトの StorageClass を確認する

Kubernetes は `StorageClass` を使用して、さまざまな種類のストレージを定義します。Minikube の初期化時に、`standard` という名前のデフォルトの `StorageClass` が自動的に作成されます。

```bash
# 現在のクラスターで利用可能なすべての StorageClass を表示する
$ kubectl get storageclass
NAME                 PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
standard (default)   k8s.io/minikube-hostpath   Delete          Immediate           false                  5d9h
```

**観察と推論:**
- デフォルトの `standard` StorageClass は、動的 PV の基盤です。PVC が作成され、この `storageClassName` を指定すると、システムは対応する PV を自動的にプロビジョニングします。

### Persistent Volume Claim (PVC) の作成とデプロイ

`dynamic-volume-pvc.yaml` という名前のファイルを作成し、その中に PVC の仕様を定義します。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc
spec:
  storageClassName: standard # standard StorageClass の使用を指定
  accessModes:
    - ReadWriteMany # 複数のノードが同時に読み書き可能
  resources:
    requests:
      storage: 2Gi
```

##### PVC のデプロイ:
`kubectl apply` コマンドを使用して、この PVC リソースを作成します。

```bash
$ kubectl apply -f dynamic-volume-pvc.yaml

$ kubectl get pvc
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
app-pvc   Bound    pvc-4600ab64-eee0-49ce-8257-b5ea2163684e   2Gi        RWX            standard       <unset>                 6s
```

### 動的にプロビジョニングされた Persistent Volume (PV) の検証

PVC が正常にデプロイされると、`StorageClass` は対応する PV を自動的に作成します。

```bash
# StorageClass によって自動的に作成された PV を表示する
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pvc-4600ab64-eee0-49ce-8257-b5ea2163684e   2Gi        RWX            Delete           Bound    default/app-pvc   standard       <unset>                          30s
```

**観察と推論:**
- 新しい PV が自動的に作成され、ステータスが `Bound` になっていることがわかります。これは、`app-pvc` PVC に正常にバインドされたことを示します。
- これは、動的 PV の中心的なメカニズム、すなわち **PVC -> StorageClass -> 自動的な PV プロビジョニング** を示しています。

### アプリケーションのデプロイとボリュームのマウント

次に、アプリケーション（例：Deployment）をデプロイし、以前に作成した PVC を Pod にマウントして、アプリケーションがそのストレージスペースを使用できるようにします。

```bash
# アプリケーションをデプロイし、PVC をマウントする
$ kubectl apply -f simple-deployment-volume.yaml

# Deployment のステータスを確認する
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

**注意:**
- `simple-deployment-volume.yaml` では、`volumes` と `volumeMounts` の設定が正しく行われ、`app-pvc` がコンテナ内の指定されたパス（例：`/app/data`）にマウントされていることを確認してください。

### ボリュームのライフサイクルとデータの永続性のテスト

データが本当に永続化されていることを確認するために、以下のテストを実行できます。

##### Pod でのデータ書き込み:
いずれかの Pod に exec し、マウントされたディレクトリにファイルを作成します。

```bash
# Pod 名を取得する
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-58449898c5-g4499   1/1     Running   0          2m57s
app-deployment-58449898c5-jxzsb   1/1     Running   0          2m57s
app-deployment-58449898c5-tb2mm   1/1     Running   0          2m57s

# Pod に exec し、テストファイルを作成する
$ kubectl exec -it [pod_name] -- touch /app/data/file002.txt
$ kubectl exec -it [pod_name] -- ls /app/data
```
この時点で、`file002.txt` が正常に作成されているはずです。

##### Pod の削除と再作成:
すべての Pod を削除し、Deployment に新しい Pod を自動的に再作成させます。

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

##### データの永続性の検証:
新しく作成された Pod に exec し、以前に書き込んだファイルが存在するかどうかを確認します。

```bash
# 新しい Pod に exec する
$ kubectl exec -it [new_pod_name] -- ls /app/data
file002.txt
```
`file002.txt` がまだ存在していることがわかります。これは、Pod が破棄されて再作成されても、データが PV によって提供されるストレージに永続化されることを証明しています。

### リソースのクリーンアップ

テストが完了したら、作成したすべてのリソースを削除してスペースを解放することを忘れないでください。

```bash
$ kubectl delete deployments --all
$ kubectl delete pvc --all
$ kubectl delete pv --all
```