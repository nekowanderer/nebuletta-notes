# Kubernetes Readiness Probe の実装

[English](../en/18_readiness_probe.md) | [繁體中文](../zh-tw/18_readiness_probe.md) | [日本語](../ja/18_readiness_probe.md) | [インデックスに戻る](../README.md)


## 概要
Readiness Probe は Pod がトラフィックを受け取る準備ができているかを確認するために使用されます。Readiness Probe が失敗した場合、Pod は Service のロードバランサーから削除されますが、Pod 自体は再起動されません。

## 前提条件

### Docker 環境の開始
```bash
$ sudo service docker start
$ docker ps
```

### Minikube クラスターの作成
```bash
$ minikube start --driver docker
$ minikube status
```

## Deployment 設定の作成

### YAML ファイルの作成
```bash
$ vi advanced-deployment.yaml
```

### Deployment 設定内容
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
        startupProbe:
          httpGet:
            path: /healthcheck
            port: 80
          initialDelaySeconds: 0
          timeoutSeconds: 1
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 30
        livenessProbe:
          httpGet:
            path: /healthcheck
            port: 80
          initialDelaySeconds: 0
          timeoutSeconds: 1
          periodSeconds: 1
          successThreshold: 1
          failureThreshold: 5
        readinessProbe:
          httpGet:
            path: /healthcheck_dependency
            port: 80
          initialDelaySeconds: 0
          timeoutSeconds: 1
          periodSeconds: 1
          successThreshold: 5
          failureThreshold: 7
```

### 設定ファイルの確認
```bash
$ cat advanced-deployment.yaml
```

## デプロイメントとモニタリング

### Deployment のデプロイ
```bash
$ kubectl apply -f advanced-deployment.yaml
```

### ReplicaSet のモニタリング
```bash
$ kubectl get rs -w
NAME                        DESIRED   CURRENT   READY   AGE
app-deployment-7f57cfb4b6   3         3         0       10s
app-deployment-7f57cfb4b6   3         3         1       15s
app-deployment-7f57cfb4b6   3         3         2       15s
app-deployment-7f57cfb4b6   3         3         3       15s
```

### Pod ステータスのモニタリング
```bash
$ kubectl get pods -w
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-7f57cfb4b6-g5sz7   1/1     Running   0          26s
app-deployment-7f57cfb4b6-mjnzw   1/1     Running   0          26s
app-deployment-7f57cfb4b6-q6xdp   1/1     Running   0          26s
app-deployment-7f57cfb4b6-q6xdp   0/1     Running   0          3m33s
```

## Service の作成

### NodePort Service のデプロイ
```bash
$ kubectl apply -f simple-service-nodeport.yaml
```

### ノード情報の取得
```bash
$ kubectl get node
NAME       STATUS   ROLES           AGE    VERSION
minikube   Ready    control-plane   2d8h   v1.33.1

$ kubectl describe node minikube | grep InternalIP
InternalIP:  192.168.49.2
```

### Service 情報の取得
```bash
$ kubectl get services
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
app-service-nodeport   NodePort    10.111.160.67   <none>        8080:30080/TCP   93s
kubernetes             ClusterIP   10.96.0.1       <none>        443/TCP          22h
```

## Readiness Probe のテスト

### ヘルスチェック失敗のシミュレーション
```bash
$ curl http://192.168.49.2:30080/healthcheck_dependency_switchstatus
[beta] served by: app-deployment-7f57cfb4b6-q6xdp.
[beta] isDependancyHealthy value switched to false.
```

### Pod ステータス変化のモニタリング
新しいターミナルウィンドウで実行：
```bash
$ kubectl get pods -w
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-7f57cfb4b6-g5sz7   1/1     Running   0          26s
app-deployment-7f57cfb4b6-mjnzw   1/1     Running   0          26s
app-deployment-7f57cfb4b6-q6xdp   1/1     Running   0          26s
app-deployment-7f57cfb4b6-q6xdp   0/1     Running   0          3m33s
```
期待する結果：`Ready 0/1` ステータスが表示されるはずです

### アプリケーションアクセスのテスト
```bash
# app-deployment-7f57cfb4b6-q6xdp からのレスポンスは表示されません
$ curl http://${NODE_IP}:${NODE_PORT}/
```

### Pod の詳細ステータス確認
```bash
$ kubectl describe pod [pod_id]
Events:
  Type     Reason     Age                 From               Message
  ----     ------     ----                ----               -------
  Normal   Scheduled  4m32s               default-scheduler  Successfully assigned default/app-deployment-7f57cfb4b6-q6xdp to minikube
  Normal   Pulling    4m32s               kubelet            Pulling image "uopsdod/k8s-hostname-amd64-beta:v1"
  Normal   Pulled     4m29s               kubelet            Successfully pulled image "uopsdod/k8s-hostname-amd64-beta:v1" in 1.456s (3.11s including waiting). Image size: 914594685 bytes.
  Normal   Created    4m29s               kubelet            Created container: app-container
  Normal   Started    4m28s               kubelet            Started container app-container
  Warning  Unhealthy  42s (x25 over 65s)  kubelet            Readiness probe failed: HTTP probe failed with statuscode: 500
```
期待する結果：`Readiness probe failed: HTTP probe failed with statuscode: 500` が表示されるはずです

## リソースのクリーンアップ

### デプロイされたリソースの削除
```bash
$ kubectl delete deployments --all
$ kubectl delete services --all
```

## 重要事項

- **Readiness Probe**：Pod がトラフィックを受け取る準備ができているかを確認
- **Liveness Probe**：Pod が生きているかを確認し、失敗時に Pod を再起動
- **Startup Probe**：Pod 起動期間中に使用され、完了後に Liveness Probe に切り替わる
- Readiness Probe が失敗した場合、Pod は Service のロードバランサーから削除されますが、再起動はされません