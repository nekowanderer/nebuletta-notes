# Kubernetes HPA (Horizontal Pod Autoscaler) 実装

[English](../en/33_k8s_hpa_in_practice.md) | [繁體中文](../zh-tw/33_k8s_hpa_in_practice.md) | [日本語](../ja/33_k8s_hpa_in_practice.md) | [インデックスに戻る](../README.md)

## 概要
このガイドでは、KubernetesでHorizontal Pod Autoscaler (HPA)を設定・使用し、CPU使用率に基づく自動スケーリングを実現する方法を説明します。

## 前提条件

### 1. Docker環境の開始
```bash
$ sudo service docker start
$ docker ps
```

### 2. Minikubeクラスタの作成
```bash
$ minikube start --driver docker
$ minikube status
```

### 3. Ingressアドオンの有効化
```bash
$ minikube addons enable ingress

$ kubectl get all -n ingress-nginx
NAME                                           READY   STATUS      RESTARTS      AGE
pod/ingress-nginx-admission-create-5fj8c       0/1     Completed   0             46h
pod/ingress-nginx-admission-patch-529hj        0/1     Completed   1             46h
pod/ingress-nginx-controller-67c5cb88f-dv4px   1/1     Running     2 (48m ago)   46h

NAME                                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx-controller             NodePort    10.102.245.181   <none>        80:30447/TCP,443:31545/TCP   46h
service/ingress-nginx-controller-admission   ClusterIP   10.96.249.142    <none>        443/TCP                      46h

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-nginx-controller   1/1     1            1           46h

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-nginx-controller-67c5cb88f   1         1         1       46h

NAME                                       STATUS     COMPLETIONS   DURATION   AGE
job.batch/ingress-nginx-admission-create   Complete   1/1           7s         46h
job.batch/ingress-nginx-admission-patch    Complete   1/1           7s         46h
```

## アプリケーションのデプロイ

### 1. HPA版アプリケーション設定の作成
既存のアプリケーション設定ファイルをコピーし、修正します：

```bash
$ cp beta-app-all.yaml beta-app-all-hpa.yaml
$ vi beta-app-all-hpa.yaml
```

### 2. アプリケーション設定ファイル内容
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app-ns
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: beta-app-deployment
  namespace: app-ns
spec:
  replicas: 1  # 1に変更
  selector:
    matchLabels:
      app: beta-app-pod
  template:
    metadata:
      labels:
        app: beta-app-pod
    spec:
      containers:
      - name: beta-app-container
        image: uopsdod/k8s-hostname-amd64-beta:v1
        ports:
        - containerPort: 80
        resources:     # このセクションを追加
          limits:
            cpu: 300m
          requests:
            cpu: 200m
---
apiVersion: v1
kind: Service
metadata:
  name: beta-app-service-clusterip
  namespace: app-ns
spec:
  type: ClusterIP      # これを追加
  selector:
    app: beta-app-pod
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
```

**重要な変更：**
- `replicas: 1`を初期レプリカ数として設定
- CPUリミットとリクエストを含む`resources`セクションを追加
- CPUリミットを`300m`、リクエストを`200m`に設定

### 3. アプリケーションのデプロイ
```bash
$ kubectl apply -f beta-app-all-hpa.yaml
namespace/app-ns created
deployment.apps/beta-app-deployment created
service/beta-app-service-clusterip created

$ kubectl get all -n app-ns
NAME                                       READY   STATUS    RESTARTS   AGE
pod/beta-app-deployment-8679d5777b-5kzjj   1/1     Running   0          9s

NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/beta-app-service-clusterip   ClusterIP   10.105.44.132   <none>        8080/TCP   9s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/beta-app-deployment   1/1     1            1           9s

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/beta-app-deployment-8679d5777b   1         1         1       9s
```

### 4. 本番アプリケーションのデプロイ
```bash
$ kubectl apply -f prod-app-all.yaml
namespace/app-ns unchanged
deployment.apps/prod-app-deployment created
service/prod-app-service-clusterip created

$ kubectl get all -n app-ns
NAME                                       READY   STATUS    RESTARTS   AGE
pod/beta-app-deployment-8679d5777b-5kzjj   1/1     Running   0          36s
pod/prod-app-deployment-586c5dcc59-b88dw   1/1     Running   0          10s
pod/prod-app-deployment-586c5dcc59-fc82v   1/1     Running   0          10s
pod/prod-app-deployment-586c5dcc59-fdtdq   1/1     Running   0          10s

NAME                                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/beta-app-service-clusterip   ClusterIP   10.105.44.132    <none>        8080/TCP   36s
service/prod-app-service-clusterip   ClusterIP   10.100.206.114   <none>        8080/TCP   10s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/beta-app-deployment   1/1     1            1           36s
deployment.apps/prod-app-deployment   3/3     3            3           10s

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/beta-app-deployment-8679d5777b   1         1         1       36s
replicaset.apps/prod-app-deployment-586c5dcc59   3         3         3       10s
```

### 5. Ingressルーティングのデプロイ
```bash
$ kubectl apply -f ingress-path.yaml

$ kubectl get ingress -n app-ns

$ kubectl get ingress -n app-ns -w
NAME           CLASS   HOSTS          ADDRESS   PORTS   AGE
ingress-path   nginx   all.demo.com             80      19s
ingress-path   nginx   all.demo.com   192.168.49.2   80      33s
```

## Metrics Serverの設定

### 1. Metrics Server設定のダウンロード
```bash
$ wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
$ mv components.yaml metrics-server.yaml
```

### 2. Metrics Server設定の修正
```bash
$ vi metrics-server.yaml
```

コンテナ引数に`--kubelet-insecure-tls`パラメータを追加：

```yaml
spec:
  containers:
  - args:
    - --cert-dir=/tmp
    - --secure-port=10250
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --kubelet-use-node-status-port
    - --metric-resolution=15s
    - --kubelet-insecure-tls  # この行を追加
    image: k8s.gcr.io/metrics-server/metrics-server:v0.7.2
```

### 3. Metrics Serverのデプロイ
```bash
$ kubectl apply -f metrics-server.yaml
$ kubectl get deployment metrics-server -n kube-system
$ kubectl get deployment metrics-server -n kube-system -w
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
metrics-server   0/1     1            0           19m
metrics-server   1/1     1            1           19m

# 起動しない場合はログを確認：
$ kubectl logs -n kube-system deployment/metrics-server
```

## HPAの設定

### 1. HPAの作成
```bash
$ kubectl autoscale deployment beta-app-deployment --cpu-percent=10 --min=1 --max=10 -n app-ns
horizontalpodautoscaler.autoscaling/beta-app-deployment autoscaled

$ kubectl get hpa -n app-ns

$ kubectl get hpa -n app-ns -w
NAME                  REFERENCE                        TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%   1         10        1          78s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%   1         10        1          2m45s
```

**HPAパラメータ説明：**
- `--cpu-percent=10`：CPU使用率がPodのCPUリクエストの10%を超えた時にスケーリングをトリガー。この例では：
  - HPA閾値 = CPUリクエスト × HPA割合 = 200 × 0.1 = 20m
- `--min=1`：最小レプリカ数は1
- `--max=10`：最大レプリカ数は10
- パラメータの詳細については下記の`CPUリソース設定説明`セクションを参照

## 自動スケーリングのテスト

### 1. スケールアップのテスト
新しいターミナルウィンドウを開く：

```bash
$ kubectl get ingress -n app-ns
$ ingress_ip=192.168.49.2
$ curl ${ingress_ip}:80/beta -H 'Host: all.demo.com'
$ while sleep 0.005; do curl -s ${ingress_ip}:80/beta -H 'Host: all.demo.com'; done
```

このコマンドはアプリケーションに継続的にリクエストを送信します。CPU使用率が10%を超えると、HPAが自動的にPodレプリカ数を増加させます。

### 2. スケールダウンのテスト
負荷テストを停止：
```bash
# Ctrl + C を押して継続リクエストを停止
```

負荷を停止した後、HPAがCPU使用率を監視し、使用率が低下すると自動的にPodレプリカ数を減少させます。

## 監視と検証

### HPAステータスの確認
```bash
$ kubectl describe hpa beta-app-deployment -n app-ns
Name:                                                  beta-app-deployment
Namespace:                                             app-ns
Labels:                                                <none>
Annotations:                                           <none>
CreationTimestamp:                                     Tue, 01 Jul 2025 14:07:28 +0000
Reference:                                             Deployment/beta-app-deployment
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  9% (19m) / 10%
Min replicas:                                          1
Max replicas:                                          10
Deployment pods:                                       3 current / 3 desired
Conditions:
  Type            Status  Reason              Message
  ----            ------  ------              -------
  AbleToScale     True    ReadyForNewScale    recommended size matches current size
  ScalingActive   True    ValidMetricFound    the HPA was able to successfully calculate a replica count from cpu resource utilization (percentage of request)
  ScalingLimited  False   DesiredWithinRange  the desired count is within the acceptable range
Events:
  Type    Reason             Age   From                       Message
  ----    ------             ----  ----                       -------
  Normal  SuccessfulRescale  62s   horizontal-pod-autoscaler  New size: 3; reason: cpu resource utilization (percentage of request) above target

# curlループを停止して約5分後：
$ kubectl get hpa -n app-ns
NAME                  REFERENCE                        TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%   1         10        1          78s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%   1         10        1          2m45s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%   1         10        1          3m
beta-app-deployment   Deployment/beta-app-deployment   cpu: 24%/10%   1         10        1          4m30s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 14%/10%   1         10        3          4m45s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 10%/10%   1         10        3          5m
beta-app-deployment   Deployment/beta-app-deployment   cpu: 9%/10%    1         10        3          5m15s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 8%/10%    1         10        3          5m45s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%    1         10        3          6m
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%    1         10        3          6m16s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%    1         10        3          6m31s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%    1         10        3          8m46s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%    1         10        3          10m
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%    1         10        1          11m


$ kubectl describe hpa beta-app-deployment -n app-ns
Name:                                                  beta-app-deployment
Namespace:                                             app-ns
Labels:                                                <none>
Annotations:                                           <none>
CreationTimestamp:                                     Tue, 01 Jul 2025 14:07:28 +0000
Reference:                                             Deployment/beta-app-deployment
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  0% (0) / 10%
Min replicas:                                          1
Max replicas:                                          10
Deployment pods:                                       3 current / 1 desired
Conditions:
  Type            Status  Reason            Message
  ----            ------  ------            -------
  AbleToScale     True    SucceededRescale  the HPA controller was able to update the target scale to 1
  ScalingActive   True    ValidMetricFound  the HPA was able to successfully calculate a replica count from cpu resource utilization (percentage of request)
  ScalingLimited  True    TooFewReplicas    the desired replica count is less than the minimum replica count
Events:
  Type    Reason             Age    From                       Message
  ----    ------             ----   ----                       -------
  Normal  SuccessfulRescale  6m17s  horizontal-pod-autoscaler  New size: 3; reason: cpu resource utilization (percentage of request) above target
  Normal  SuccessfulRescale  1s     horizontal-pod-autoscaler  New size: 1; reason: All metrics below target
```

## リソースのクリーンアップ

### 全リソースのクリーンアップ
```bash
$ kubectl delete namespace app-ns
```

## HPA作成方法の説明

### この実験のHPA作成方法
この実験では`kubectl autoscale`コマンドを使用してHPAを作成し、Kubernetesの[デフォルト動作](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#default-behavior)設定を使用：
- **スケールダウン安定化時間**：5分 (`stabilizationWindowSeconds: 300`)
- **スケールアップ安定化時間**：即座 (`stabilizationWindowSeconds: 0`)

### カスタムHPA設定
HPA動作を調整する必要がある場合、カスタムHPA YAMLファイルを作成できます：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: beta-app-deployment
  namespace: app-ns
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: beta-app-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 10
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 600  # カスタムスケールダウン安定化時間
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 180  # カスタムスケールアップ安定化時間
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
```

### 既存HPAの修正
patchコマンドを使用して既存のHPAを修正することも可能：

```bash
$ kubectl patch hpa beta-app-deployment -n app-ns --type='merge' -p='
spec:
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 60
'
```

## CPUリソース設定の説明

#### CPU単位の説明
- **1 CPU = 1000m (millicores)**
- **300m = 0.3 CPU = 単一CPUコアの30%**
- **200m = 0.2 CPU = 単一CPUコアの20%**

#### CPU Requests（リクエスト） - 200m
```yaml
resources:
  requests:
    cpu: 200m
```

**意味：**
- **リソース保証**：KubernetesがPodに最低200mのCPUリソースを割り当てることを保証
- **スケジューリング基準**：スケジューラは十分なCPUリソースを持つノードにのみPodをスケジュール
- **HPA計算基準**：HPAはrequests値をCPU使用率パーセンテージの計算基準として使用
  - なぜこの設計？
    - 相対性：絶対値ではなくパーセンテージを使用し、HPAを異なるサイズのPodに適応
    - 柔軟性：同じHPA設定を異なるリソース設定のアプリケーションに適用可能
    - 予測可能性：requestsベースの計算で一貫した動作を保証
- **リソース予約**：PodがCPUを使用していなくても、これらのリソースは予約され、他のPodには割り当てられない

#### CPU Limits（制限） - 300m
```yaml
resources:
  limits:
    cpu: 300m
```

**意味：**
- **リソース上限**：Podは最大300mのCPUリソースしか使用できない
- **スロットリング機能**：Podが300mを超えるCPUを使用しようとすると、スロットリングされる
- **リソース濫用防止**：単一PodによるCPUリソースの過度な消費を防ぎ、他のPodへの影響を回避
- **バースト使用**：Podは短時間でrequestsを超えるがlimits未満のリソースを使用可能

#### 実際の運用例
- requests: 200m
- limits: 300m

| 負荷 | CPU使用量 | 計算結果 | HPA動作 | システム状態 |
|------|-----------|----------|----------|--------------|
| 🔵 **低** | 50m CPU | 50m ÷ 200m = 25% | HPAスケールダウンをトリガーしない | ノードに150m CPUが予約されているが未使用 |
| 🟢 **通常** | 150m CPU | 150m ÷ 200m = 75% | 閾値が70%の場合、スケールアップをトリガー | Pod正常動作、リソース十分 |
| 🔴 **高** | 400m CPU使用を試行 | 実際には300mのみ使用可能（limits制限） | 超過分はスロットリング、パフォーマンス低下の可能性 | Podパフォーマンス制限、limits調整が必要 |

#### 設定推奨事項

**Requests設定原則：**
- アプリケーションの平均CPU使用率に設定
- 低すぎる：Podが十分なリソースを得られない可能性
- 高すぎる：ノードリソースの浪費、クラスタ利用率の低下

**Limits設定原則：**
- 通常requestsの1.5-2倍に設定
- 低すぎる：アプリケーションパフォーマンス制限の可能性
- 高すぎる：リソース保護の効果を失う

**HPA考慮事項：**
- HPAはrequestsベースでパーセンテージを計算
- `--cpu-percent=10`の設定は、CPU使用率がrequestsの10%を超えた時にスケールアップをトリガー
- この例では、CPU使用率が20m（200m × 10%）を超えた時にスケールアップをトリガー

## 重要な注意事項

1. **Metrics Serverの正常動作が必要**：HPAはMetrics ServerからCPUとメモリ使用率メトリクスを取得
2. **リソース制限設定**：Deploymentで`resources.requests`と`resources.limits`を設定する必要がある
3. **CPU パーセンテージ計算**：HPAは`requests`値をベースにCPU使用率パーセンテージを計算
4. **クールダウン時間**：HPAには内蔵のクールダウン時間（約5分）があり、頻繁なスケーリング操作を回避
5. **監視間隔**：デフォルトで15秒ごとにメトリクスをチェック

## トラブルシューティング

### よくある問題
1. **HPAがメトリクスを取得できない**：Metrics Serverが正常に動作しているか確認
2. **スケーリングが機能しない**：Deploymentでリソース制限が設定されているか確認
3. **Ingressにアクセスできない**：Ingressコントローラの状態とネットワーク設定を確認