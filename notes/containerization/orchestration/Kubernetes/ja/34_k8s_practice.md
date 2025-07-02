# Kubernetes 実践演習

[English](../en/34_k8s_practice.md) | [繁體中文](../zh-tw/34_k8s_practice.md) | [日本語](../ja/34_k8s_practice.md) | [インデックスに戻る](../README.md)

## 事前準備

### 1. 既存環境のクリーンアップ
```bash
$ minikube stop
$ minikube delete
$ minikube status
```

### 2. 新しいMinikubeクラスタの作成
```bash
$ minikube start --driver docker
```

---

## テンプレート設計フェーズ

### フェーズ1：Nginx Deployment

**要件：**
- Podテンプレートのイメージ名として `nginx:1.29`（2025/07時点）を使用
- Deployment名を `nginx-app-deployment` として定義
- 5個のRunning Podを維持
- ラベルの使用に注意

```bash
$ vi nginx-deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app-deployment
spec:
  replicas: 5
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
        image: nginx:1.29
        ports: 
        - containerPort: 80
```

**検証：**
**期待される結果：** Ready状態「5/5」で5個のRunning Podが確認できる
```bash
$ kubectl get deployments nginx-app-deployment
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
nginx-app-deployment   5/5     5            5           19s
```

---

### フェーズ2：Nginx Service

**要件：**
- NodePortタイプを使用
- Service名を `nginx-app-service-nodeport` として定義
- このServiceを通じてNginx Deployment内のPodのネットワークリクエストを管理
- 外部接続用ポートとして8081を使用
- ラベルの使用に注意

**注意事項：**
- Nginxアプリケーションはポート80でリッスン

```bash
$ vi nginx-service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-app-service-nodeport
spec:
  type: NodePort
  selector:
    app: app-pod
  ports:
    - protocol: TCP
      port: 8081
      targetPort: 80
      nodePort: 30080
```

**検証：**
**期待される結果：** NodePortタイプ、ポートが8081:3xxxxと表示される
```bash
$ kubectl get services nginx-app-service-nodeport
NAME                         TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
nginx-app-service-nodeport   NodePort   10.107.180.108   <none>        8081:30080/TCP   27s
```

**期待される結果：** Nginxホームページ情報が表示される（例：`<h1>Welcome to nginx!</h1>`）
**注意：** node_ipはminikubeが配置されているノードIP、つまりコンテナIPを指す
```bash
ノードIPの取得
$ kubectl describe node minikube | grep InternalIP
  InternalIP:  192.168.49.2

$ curl 192.168.49.2:30080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

---

### フェーズ3：Apache Deployment

**要件：**
- Docker練習で作成したカスタム `your_dockerhub_account/xxx_image:latest` をPodテンプレートのイメージとして使用

```bash 
$ vi Dockerfile
```

```Dockerfile
FROM alpine:3.16
RUN apk --update add apache2
RUN rm -rf /var/cache/apk/*
ENTRYPOINT ["httpd", "-D", "FOREGROUND"]
```
- 自分でビルドしてDocker Hubにプッシュすることを忘れずに

- Deployment名を `apache-app-deployment` として定義
- 5個のRunning Podを維持
- ラベルの使用に注意

```bash
$ vi apache-deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apache-app-deployment
spec:
  replicas: 5
  selector:
    matchLabels:
      app: apache-pod
  template:
    metadata:
      labels:
        app: apache-pod
    spec:
      containers:
      - name: app-container
        image: nekowandrer/test-img:latest
        ports:
        - containerPort: 80
```

**注意事項：**
- `your_dockerhub_account`を自分のDockerHubアカウントに置き換えることを忘れずに
- Apacheアプリケーションはポート80でリッスン

**検証：**
**期待される結果：** Ready状態「5/5」で5個のRunning Podが確認できる
```bash
$ kubectl get deployments apache-app-deployment
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
apache-app-deployment   5/5     5            5           20s
```

---

### フェーズ4：Apache Service

**要件：**
- NodePortタイプを使用
- Service名を `apache-app-service-nodeport` として定義
- このServiceを通じてApache Deployment内のPodのネットワークリクエストを管理
- 外部接続用ポートとして8082を使用
- ラベルの使用に注意

```bash
$ vi apache-service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: apache-app-service-nodeport
spec:
  type: NodePort
  selector:
    app: apache-pod
  ports:
    - protocol: TCP
      port: 8082
      targetPort: 80
      nodePort: 30081
```

**検証：**
**期待される結果：** NodePortタイプ、ポートが8082:3xxxxと表示される
```bash
$ kubectl get services apache-app-service-nodeport
NAME                          TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
apache-app-service-nodeport   NodePort   10.97.29.96   <none>        8082:30081/TCP   17s
```

```bash
$ curl [node_ip]:[node_exposed_port]
```
**期待される結果：** Apacheホームページ情報が表示される
**注意：** node_ipはminikubeが配置されているノードIP、つまりコンテナIPを指す

```bash
$ curl 192.168.49.2:30081
<html><body><h1>It works!</h1></body></html>
```

---

### フェーズ5：Ingress

**要件：**
- まず `minikube addons enable ingress` を実行してnginx ingress controllerを有効化
- `nginx.demo.com` からのリクエストをNginx Serviceに送るルールを作成
- `apache.demo.com` からのリクエストをApache Serviceに送るルールを作成

```bash
$ vi hostname-ingress.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-hostname
spec:
  ingressClassName: nginx
  rules:
  - host: nginx.demo.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: nginx-app-service-nodeport
            port:
              number: 8081
  - host: apache.demo.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: apache-app-service-nodeport
            port:
              number: 8082
```

**検証：**
**期待される結果：** しばらく待つと、ingress IP、2つのホスト名、およびポート80が表示される
```bash
$ kubectl get ingress -w
NAME               CLASS   HOSTS                            ADDRESS        PORTS   AGE
ingress-hostname   nginx   nginx.demo.com,apache.demo.com   192.168.49.2   80      7m50s
```

**期待される結果：** Nginxホームページ情報が表示される
```bash
$ curl 192.168.49.2:80 -H 'Host: nginx.demo.com'
```

**期待される結果：** Apacheホームページ情報が表示される
```bash
$ curl 192.168.49.2:80 -H 'Host: apache.demo.com'
```

---

## 実装検証

### 環境パラメータ設定
すべて完了後、以下の環境パラメータを設定：
```bash
NODE_IP=192.168.xxx.xxx
NODE_PORT_NGINX=3xxxx
NODE_PORT_APACHE=3xxxx
INGRESS_IP=192.168.xxx.xxx
```

### 検証1：リソース状態
環境パラメータ設定後、以下のコマンドを実行してスクリーンショットを撮る：
```bash
$ kubectl get pods
NAME                                     READY   STATUS    RESTARTS   AGE
apache-app-deployment-5cc5db7485-44dwg   1/1     Running   0          17m
apache-app-deployment-5cc5db7485-9t9z6   1/1     Running   0          17m
apache-app-deployment-5cc5db7485-dv2db   1/1     Running   0          17m
apache-app-deployment-5cc5db7485-lfh2v   1/1     Running   0          17m
apache-app-deployment-5cc5db7485-wq6vc   1/1     Running   0          17m
nginx-app-deployment-86cf7c4ddb-b7zm4    1/1     Running   0          33m
nginx-app-deployment-86cf7c4ddb-dzfrv    1/1     Running   0          33m
nginx-app-deployment-86cf7c4ddb-jkwsj    1/1     Running   0          33m
nginx-app-deployment-86cf7c4ddb-mxvcz    1/1     Running   0          33m
nginx-app-deployment-86cf7c4ddb-sgdbg    1/1     Running   0          33m

$ kubectl get rs
NAME                               DESIRED   CURRENT   READY   AGE
apache-app-deployment-5cc5db7485   5         5         5       17m
nginx-app-deployment-86cf7c4ddb    5         5         5       33m

$ kubectl get deployments
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
apache-app-deployment   5/5     5            5           17m
nginx-app-deployment    5/5     5            5           33m

$ kubectl get services
NAME                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
apache-app-service-nodeport   NodePort    10.97.29.96      <none>        8082:30081/TCP   15m
kubernetes                    ClusterIP   10.96.0.1        <none>        443/TCP          9d
nginx-app-service-nodeport    NodePort    10.107.180.108   <none>        8081:30080/TCP   29m

$ kubectl get ingress
NAME               CLASS   HOSTS                            ADDRESS        PORTS   AGE
ingress-hostname   nginx   nginx.demo.com,apache.demo.com   192.168.49.2   80      11m
```

### 検証2：Nginxサービステスト
環境パラメータ設定後、以下のコマンドを実行してスクリーンショットを撮る：
```bash
$ curl ${NODE_IP}:${NODE_PORT_NGINX}
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

$ curl ${INGRESS_IP}:80 -H 'Host: nginx.demo.com'
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

### 検証3：Apacheサービステスト
環境パラメータ設定後、以下のコマンドを実行してスクリーンショットを撮る：
```bash
$ curl ${NODE_IP}:${NODE_PORT_APACHE}
<html><body><h1>It works!</h1></body></html>

$ curl ${INGRESS_IP}:80 -H 'Host: apache.demo.com'
<html><body><h1>It works!</h1></body></html>
```

### 検証ファイル一覧
最終的に、yamlファイルは合計5つになる：

1. `nginx-deployment.yaml`
2. `nginx-service.yaml`
3. `apache-deployment.yaml`
4. `apache-service.yaml`
5. `hostname-ingress.yaml`