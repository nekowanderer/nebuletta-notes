# Kubernetes Service LoadBalancer 実装ガイド

[English](../en/15_k8s_loadbalancer_mode.md) | [繁體中文](../zh-tw/15_k8s_loadbalancer_mode.md) | [日本語](../ja/15_k8s_loadbalancer_mode.md) | [インデックスに戻る](../README.md)

## 概要
LoadBalancerタイプのServiceは、Kubernetesで最も一般的に使用される外部アクセス方法の一つです。バックエンドのPodにトラフィックを分散するための外部ロードバランサーを自動的に作成します。

## 前提条件

##### 1. Dockerサービスの起動
```bash
$ sudo service docker start
$ docker ps
```

##### 2. Minikubeクラスターの作成
```bash
$ minikube start --driver docker
$ minikube status
```

## アプリケーションのデプロイ

[10_deploy_deployments](./10_deploy_deployments.md)の`simple-deployment.yaml`を再利用

##### 1. Deploymentのデプロイ
```bash
$ kubectl apply -f simple-deployment.yaml
$ kubectl get deployments
```

##### 2. LoadBalancer Serviceの作成

`simple-service-loadbalancer.yaml`ファイルを作成：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: app-pod
  ports:
    - protocol: TCP
      port: 8080        # Serviceが公開するポート
      targetPort: 80    # Pod内部のポート
```

##### 3. Serviceのデプロイ
```bash
$ kubectl apply -f simple-service-loadbalancer.yaml

$ kubectl get services
NAME                       TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
app-service-loadbalancer   LoadBalancer   10.110.110.138   <pending>     8080:31830/TCP   6s  # 注意：EXTERNAL-IPがpending状態
app-service-nodeport       NodePort       10.106.158.207   <none>        8080:30080/TCP   26m
kubernetes                 ClusterIP      10.96.0.1        <none>        443/TCP          26h

$ kubectl describe service app-service-loadbalancer
Name:                     app-service-loadbalancer
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=app-pod
Type:                     LoadBalancer
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.110.110.138
IPs:                      10.110.110.138
Port:                     <unset>  8080/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  31830/TCP
Endpoints:                10.244.0.22:80,10.244.0.21:80,10.244.0.20:80
Session Affinity:         None
External Traffic Policy:  Cluster
Internal Traffic Policy:  Cluster
Events:                   <none>
```

## 外部アクセスの有効化

##### 1. Minikube Tunnelの起動
別のターミナルウィンドウで実行：
```bash
$ minikube tunnel
Status:
        machine: minikube
        pid: 25727
        route: 10.96.0.0/12 -> 192.168.49.2
        minikube: Running
        services: [app-service-loadbalancer]
    errors:
                minikube: no errors
                router: no errors
                loadbalancer emulator: no errors
```

このコマンドはネットワークトンネルを作成し、ローカルでロードバランサーをシミュレートしてminikubeクラスターにトラフィックを導きます。

##### 2. ロードバランシング機能のテスト
```bash
# サービスステータスの確認
$ kubectl get services
NAME                       TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)          AGE
app-service-loadbalancer   LoadBalancer   10.110.110.138   10.110.110.138   8080:31830/TCP   2m23s # EXTERNAL-IPがpendingではなくなり、バインドされました
app-service-nodeport       NodePort       10.106.158.207   <none>           8080:30080/TCP   28m
kubernetes                 ClusterIP      10.96.0.1        <none>           443/TCP          26h

# 接続テスト（ロードバランシング効果を観察するため繰り返し実行）
$ curl [service_external_ip]:8080
$ curl [service_external_ip]:8080
$ curl [service_external_ip]:8080
$ curl [service_external_ip]:8080
$ curl [service_external_ip]:8080
```

## リソースのクリーンアップ

##### 1. 全リソースの削除
```bash
$ kubectl delete all --all
pod "app-deployment-8687d78656-d6x8t" deleted
pod "app-deployment-8687d78656-gp96l" deleted
pod "app-deployment-8687d78656-p7k6j" deleted
service "app-service-loadbalancer" deleted
service "app-service-nodeport" deleted
service "kubernetes" deleted
deployment.apps "app-deployment" deleted
```

##### 2. Minikube Tunnelの停止
`minikube tunnel`を実行しているターミナルで`Ctrl + C`を押す

## 重要な概念

- **LoadBalancerタイプ**：外部ロードバランサーを自動的に作成
- **port**：Serviceが公開するポート
- **targetPort**：Pod内部で実際にリッスンしているポート
- **Minikube Tunnel**：ローカル環境でクラウドロードバランサー機能をシミュレート

## 注意事項

1. LoadBalancerタイプはクラウドプロバイダーのサポートが必要
2. Minikube環境では外部アクセスをシミュレートするために`minikube tunnel`が必要
3. 実際の本番環境では、LoadBalancerが自動的に外部IPアドレスを割り当てます 