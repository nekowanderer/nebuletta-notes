# Kubernetes Service NodePort 実装ガイド

[English](../en/14_k8s_nodeport_mode.md) | [繁體中文](../zh-tw/14_k8s_nodeport_mode.md) | [日本語](../ja/14_k8s_nodeport_mode.md) | [インデックスに戻る](../README.md)

## 概要

NodePortは、Kubernetes Serviceの一種で、各ノード上で特定のポート（範囲30000-32767）を開き、外部トラフィックがノードIPとそのポートを通じてクラスター内のサービスにアクセスできるようにします。

## 前提条件

##### 1. Docker環境の起動
```bash
# Dockerサービスを起動
$ sudo service docker start

# Dockerが実行中であることを確認
$ docker ps
```

##### 2. Minikubeクラスターの作成
```bash
# Docker driverでMinikubeを起動
$ minikube start --driver docker

# クラスターの状態を確認
$ minikube status
```

## アプリケーションのデプロイ

[10_deploy_deployments](./10_deploy_deployments.md)の`simple-deployment.yaml`を再利用

##### 1. Deploymentのデプロイ
```bash
# アプリケーションをデプロイ
$ kubectl apply -f simple-deployment.yaml

# デプロイメントの状態を確認
$ kubectl get deployments
```

## NodePort Serviceの作成

##### 1. Service設定ファイルの作成
`simple-service-nodeport.yaml`ファイルを作成：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service-nodeport
spec:
  type: NodePort
  selector:
    app: app-pod
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
      nodePort: 30080  # オプション：特定のNodePortを指定、範囲30000-32767
```

##### 2. Serviceのデプロイ
```bash
# NodePort Serviceをデプロイ
$ kubectl apply -f simple-service-nodeport.yaml

# すべてのサービスを表示
$ kubectl get services
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
app-service-nodeport   NodePort    10.106.158.207   <none>        8080:30080/TCP   5s  # <- 30080がnode port
kubernetes             ClusterIP   10.96.0.1        <none>        443/TCP          25h

# サービスの詳細を表示
$ kubectl describe service app-service-nodeport
Name:                     app-service-nodeport
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=app-pod
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.106.158.207
IPs:                      10.106.158.207
Port:                     <unset>  8080/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  30080/TCP
Endpoints:                10.244.0.20:80,10.244.0.22:80,10.244.0.21:80
Session Affinity:         None
External Traffic Policy:  Cluster
Internal Traffic Policy:  Cluster
Events:                   <none>
```

## ノード情報の取得

##### 1. ノード情報の表示
```bash
# すべてのノードを表示
$ kubectl get nodes
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   25h   v1.33.1

# ノードの詳細を表示してノードIPを取得
$ kubectl describe node minikube
...
ddresses:
  InternalIP:  192.168.49.2 # <- これをコピー、これがノードIP
  Hostname:    minikube
...
```

##### 2. ノードIPの取得
`kubectl describe node minikube`の出力から、`InternalIP`または`ExternalIP`フィールドを見つける（この例では`InternalIP`）。

## NodePort接続のテスト

##### 1. ノードIPとNodePortを使用したテスト
```bash
# ノードIPとNodePortを使用してテスト
# 形式：curl [node_ip]:[nodeport]
$ curl [node_ip]:30080
```

##### 2. MinikubeのサービスURLを使用
```bash
# Minikubeは便利なサービスURL機能を提供
$ minikube service app-service-nodeport --url

# 返されたURLを使用してテスト
$ curl $(minikube service app-service-nodeport --url)
```

## 重要な概念

##### NodePortの特徴
- **外部アクセス**：ノードIPとNodePortを通じてクラスター外部からサービスにアクセス可能
- **ポート範囲**：NodePortは30000-32767の範囲のポートを使用
- **ノードレベル**：各ノードが同じNodePortを開く
- **負荷分散**：kube-proxyが自動的にバックエンドポッドにトラフィックを分散

##### ネットワークフロー
```
外部クライアント → ノードIP:NodePort → kube-proxy → Pod
```

##### 使用場面
- 開発環境でのテスト
- 内部ネットワークアクセス
- クラウドロードバランサーが不要なシナリオ
- 特定ノードへのアクセス要件

## 高度な設定

##### 1. NodePortの指定
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service-nodeport
spec:
  type: NodePort
  selector:
    app: app-pod
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
      nodePort: 30080  # 特定のNodePortを指定
```

##### 2. External Traffic Policy
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service-nodeport
spec:
  type: NodePort
  externalTrafficPolicy: Local  # またはCluster
  selector:
    app: app-pod
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
```

## トラブルシューティング

##### よくある問題
1. **外部から接続できない**
   - ファイアウォール設定でNodePort範囲のトラフィックが許可されていることを確認
   - ノードIPが正しいことを確認
   - ポッドが実行中で健全であることを確認

2. **NodePortの競合**
   - 他のサービスが同じNodePortを使用していないか確認
   - 手動で異なるNodePortを指定可能

3. **ポート範囲の問題**
   - NodePortは30000-32767の範囲内である必要がある
   - システム予約ポートの使用を避ける

##### デバッグコマンド
```bash
# サービスの状態を確認
$ kubectl get services

# エンドポイントを確認
$ kubectl get endpoints app-service-nodeport

# ポッドの状態を確認
$ kubectl get pods -l app=app-pod

# サービスの詳細を表示
$ kubectl describe service app-service-nodeport
```

## 関連リソース

- [Kubernetes Service公式ドキュメント](https://kubernetes.io/docs/concepts/services-networking/service/)
- [NodePort Serviceタイプの説明](./11_k8s_service_types.md)
- [KubernetesでのDNATとSNATの動作](./12_dnat_and_snat_in_k8s.md) 