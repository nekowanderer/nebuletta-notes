# Kubernetes Startup および Liveness Probe の実装

[English](../en/17_startup_liveness_probes.md) | [繁體中文](../zh-tw/17_startup_liveness_probes.md) | [日本語](../ja/17_startup_liveness_probes.md) | [インデックスに戻る](../README.md)

このドキュメントでは、KubernetesでStartup ProbeとLiveness Probeを実装し、実際の操作を通してプローブの動作を観察する方法を示します。

### 前提準備

##### 1. Docker環境の起動
```bash
$ sudo service docker start
$ docker ps
```

##### 2. Minikube Clusterの作成
```bash
$ minikube start --driver docker
$ minikube status
```

### Deployment設定の作成

[10_deploy_deployments](./10_deploy_deployments.md)のsimple-deployment.ymlを再利用します

##### 1. 基本設定のコピー
```bash
$ cp simple-deployment.yaml advanced-deployment.yaml
```

##### 2. 設定ファイルの編集
`vi advanced-deployment.yaml`を使用してファイルを編集し、Startup ProbeとLiveness Probeの設定を追加します：

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
        imagePullPolicy: Always
        ports:
        - containerPort: 80
        
        # Startup Probe: アプリケーションに起動時間を与える
        startupProbe:
          httpGet:
            path: /healthcheck
            port: 80
          initialDelaySeconds: 0
          timeoutSeconds: 1
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 30
        
        # Liveness Probe: アプリケーションの生存を監視
        livenessProbe:
          httpGet:
            path: /healthcheck
            port: 80
          initialDelaySeconds: 0
          timeoutSeconds: 1
          periodSeconds: 1
          successThreshold: 1
          failureThreshold: 5
```

##### 3. 設定の確認
```bash
$ cat advanced-deployment.yaml
```

### デプロイメントとモニタリング

##### 1. Deploymentのデプロイ
```bash
$ kubectl apply -f advanced-deployment.yaml
```

##### 2. ReplicaSetの監視
```bash
$ kubectl get rs -w
NAME                        DESIRED   CURRENT   READY   AGE
app-deployment-6786b5648d   3         3         0       8s
app-deployment-6786b5648d   3         3         1       11s
app-deployment-6786b5648d   3         3         2       11s
app-deployment-6786b5648d   3         3         3       11s
```

##### 3. Serviceのデプロイ
```bash
$ kubectl apply -f simple-service-nodeport.yaml
```

### プローブ機能のテスト

##### 1. ノード情報の取得
```bash
$ kubectl get node
$ kubectl describe node minikube
...
Addresses:
  InternalIP:  192.168.49.2
  Hostname:    minikube
...

# ノードIPの設定（実際の出力に応じて調整）
$ NODE_IP=192.168.49.2
```

##### 2. サービス情報の取得
```bash
$ kubectl get services
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
app-service-nodeport   NodePort    10.104.200.155   <none>        8080:30080/TCP   54s
kubernetes             ClusterIP   10.96.0.1        <none>        443/TCP          7h54m

# NodePortの設定（実際の出力に応じて調整）
$ NODE_PORT=3XXXX
```

##### 3. ヘルスチェック失敗のシミュレート
```bash
$ curl http://${NODE_IP}:${NODE_PORT}/healthcheck_switchstatus
[beta] served by: app-deployment-6786b5648d-757bq.
[beta] isHealthy value switched to false.
```

##### 4. Pod状態の監視
```bash
$ kubectl get pods -w
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-6786b5648d-757bq   1/1     Running   0          3m42s
app-deployment-6786b5648d-jlvdw   1/1     Running   0          3m42s
app-deployment-6786b5648d-psqph   1/1     Running   0          3m42s
app-deployment-6786b5648d-757bq   0/1     Running   1 (2s ago)   4m9s
app-deployment-6786b5648d-757bq   0/1     Running   1 (3s ago)   4m10s
app-deployment-6786b5648d-757bq   1/1     Running   1 (3s ago)   4m10s

# 特定Podの詳細情報を確認
$ kubectl describe pod [pod_id]
...
Events:
  Type     Reason     Age                  From               Message
  ----     ------     ----                 ----               -------
  Normal   Scheduled  4m36s                default-scheduler  Successfully assigned default/app-deployment-6786b5648d-757bq to minikube
  Normal   Pulled     4m35s                kubelet            Successfully pulled image "uopsdod/k8s-hostname-amd64-beta:v1" in 1.663s (1.663s including waiting). Image size: 914594685 bytes.
  Warning  Unhealthy  60s (x5 over 64s)    kubelet            Liveness probe failed: HTTP probe failed with statuscode: 500
  Normal   Killing    60s                  kubelet            Container app-container failed liveness probe, will be restarted
  Normal   Pulling    30s (x2 over 4m36s)  kubelet            Pulling image "uopsdod/k8s-hostname-amd64-beta:v1"
  Normal   Created    28s (x2 over 4m35s)  kubelet            Created container: app-container
  Normal   Started    28s (x2 over 4m35s)  kubelet            Started container app-container
  Normal   Pulled     28s                  kubelet            Successfully pulled image "uopsdod/k8s-hostname-amd64-beta:v1" in 1.462s (1.462s including waiting). Image size: 914594685 bytes.
...
```

### リソースのクリーンアップ

```bash
$ kubectl delete deployments --all
$ kubectl delete services --all
```

### プローブ設定の説明

##### Startup Probeパラメータ
- **`initialDelaySeconds: 0`**: コンテナ起動後すぐにプローブを開始
- **`timeoutSeconds: 1`**: 各プローブのタイムアウト時間は1秒
- **`periodSeconds: 10`**: 10秒毎にプローブを実行
- **`successThreshold: 1`**: 1回成功すれば起動完了と判定
- **`failureThreshold: 30`**: 最大30回の失敗を許可（合計5分の起動時間）

##### Liveness Probeパラメータ
- **`initialDelaySeconds: 0`**: コンテナ起動後すぐにプローブを開始
- **`timeoutSeconds: 1`**: 各プローブのタイムアウト時間は1秒
- **`periodSeconds: 1`**: 1秒毎にプローブを実行
- **`successThreshold: 1`**: 1回成功すれば生存と判定
- **`failureThreshold: 5`**: 最大5回の失敗を許可（5秒後にコンテナを再起動）

### 期待される結果

1. **起動段階**: Startup Probeがアプリケーションに十分な起動時間を提供
2. **実行段階**: Liveness Probeがアプリケーションの状態を継続的に監視
3. **障害処理**: ヘルスチェックが失敗した際、Kubernetesが自動的にコンテナを再起動
4. **監視検証**: `kubectl describe pod`でプローブの詳細な実行記録を確認可能