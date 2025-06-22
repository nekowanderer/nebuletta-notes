# Kubernetes Service ClusterIP 実装ガイド

[English](../en/13_k8s_clusterip_mode.md) | [繁體中文](../zh-tw/13_k8s_clusterip_mode.md) | [日本語](../ja/13_k8s_clusterip_mode.md) | [インデックスに戻る](../README.md)

## 概要

ClusterIPは、Kubernetesで最も基本的なServiceタイプであり、主にクラスター内のPod間の通信に使用されます。このガイドでは、ClusterIP Serviceの実際のデプロイとテスト方法を説明します。

## 前提条件

### 1. Docker環境の起動
```bash
# Dockerサービスの起動
$ sudo service docker start

# Dockerが実行中であることを確認
$ docker ps
```

### 2. Minikubeクラスターの作成
```bash
# Docker driverでMinikubeを起動
$ minikube start --driver docker

# クラスターの状態を確認
$ minikube status
```

## アプリケーションのデプロイ

[10_deploy_deployments](./10_deploy_deployments.md)の`simple-deployment.yaml`を再利用

### 1. Deploymentのデプロイ
```bash
# アプリケーションのデプロイ
$ kubectl apply -f simple-deployment.yaml

# Deploymentの状態を確認
$ kubectl get deployments
```

## ClusterIP Serviceの作成

### 1. Service設定ファイルの作成
`simple-service-clusterip.yaml`ファイルを作成：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service-clusterip
spec:
  type: ClusterIP
  selector:
    app: app-pod
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
```

### 2. Serviceのデプロイ
```bash
# ClusterIP Serviceのデプロイ
$ kubectl apply -f simple-service-clusterip.yaml

# すべてのServiceを表示
$ kubectl get services
NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
app-service-clusterip   ClusterIP   10.105.138.111   <none>        8080/TCP   7s
kubernetes              ClusterIP   10.96.0.1        <none>        443/TCP    25h

# Serviceの詳細情報を表示
$ kubectl describe service app-service-clusterip
Name:                     app-service-clusterip
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=app-pod
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.105.138.111
IPs:                      10.105.138.111
Port:                     <unset>  8080/TCP
TargetPort:               80/TCP
Endpoints:                10.244.0.22:80,10.244.0.21:80,10.244.0.20:80
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>
```

## ClusterIP接続のテスト

### 1. Service IPの取得
```bash
# Service情報を表示してClusterIPを取得
$ kubectl get services
```

### 2. Minikubeコンテナからのテスト
```bash
# Dockerコンテナを表示
$ docker ps

# Minikubeコンテナに入る
$ docker exec -it minikube bash

# Service IPを使用して接続をテスト（[service_ip]を実際のClusterIPに置き換え）
$ curl [service_ip]:8080

# コンテナから出る
$ exit
```

## 重要な概念

### ClusterIPの特徴
- **クラスター内通信**: Kubernetesクラスター内でのみアクセス可能
- **自動負荷分散**: kube-proxyがバックエンドPodにトラフィックを自動的に分散
- **DNS解決**: クラスター内でService名をDNSで解決可能
- **固定IP**: Podが再起動してもService IPはクラスター内で固定

### ネットワークフロー
```
Pod A → ClusterIP Service → Pod B
```

### 使用場面
- マイクロサービス間の内部通信
- データベース接続
- 内部API呼び出し
- 監視・ログ収集サービス

## トラブルシューティング

### よくある問題
1. **Serviceに接続できない**
   - PodラベルがService selectorと一致するか確認
   - Podが実行中で健全であることを確認

2. **ポートの不一致**
   - Serviceの`targetPort`がPodコンテナポートと一致することを確認

3. **DNS解決の問題**
   - クラスター内で完全なDNS名を使用: `service-name.namespace.svc.cluster.local`

## 関連リソース

- [Kubernetes Service公式ドキュメント](https://kubernetes.io/docs/concepts/services-networking/service/)
- [ClusterIP Serviceタイプの説明](../11_k8s_service_types.md) 