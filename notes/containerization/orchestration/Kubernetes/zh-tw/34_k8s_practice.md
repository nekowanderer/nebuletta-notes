# Kubernetes 實作練習

[English](../en/34_k8s_practice.md) | [繁體中文](../zh-tw/34_k8s_practice.md) | [日本語](../ja/34_k8s_practice.md) | [回到索引](../README.md)

## 前置作業

### 1. 清理現有環境
```bash
$ minikube stop
$ minikube delete
$ minikube status
```

### 2. 建立全新 Minikube Cluster
```bash
$ minikube start --driver docker
```

---

## 模板設計階段

### 階段一：Deployments Nginx

**要求：**
- 使用 `nginx:1.29` (as of 2025/07) 為 Pod 模板中的 Image 名稱
- 將 Deployment 名稱定義為 `nginx-app-deployment`
- 維持 5 個 Running Pods
- 注意 labels 的使用

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

**驗證：**
**預期結果：** 看見 5 個 Running Pods 使狀態成為 Ready "5/5"
```bash
$ kubectl get deployments nginx-app-deployment
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
nginx-app-deployment   5/5     5            5           19s
```

---

### 階段二：Service Nginx

**要求：**
- 使用 NodePort 類別
- 將 Service 名稱定義為 `nginx-app-service-nodeport`
- 將 Deployments Nginx 中的 Pods 交由此 Service 管理網路請求
- 使用 8081 作為開放外界連線的 Port
- 注意 labels 的使用

**注意事項：**
- Nginx 應用程式監聽 80 port

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

**驗證：**
**預期結果：** 看見 NodePort 類別，Port 為 8081:3xxxx
```bash
$ kubectl get services nginx-app-service-nodeport
NAME                         TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
nginx-app-service-nodeport   NodePort   10.107.180.108   <none>        8081:30080/TCP   27s
```

**預期結果：** 看見 Nginx 首頁資訊 (ex. `<h1>Welcome to nginx!</h1>`)
**提醒：** node_ip 指的是 minikube 所處在的 node ip，也就是所處在的 container ip
```bash
取得 node ip
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

### 階段三：Deployments Apache

**要求：**
- 使用在 Docker 練習中，所客製化建立的 `your_dockerhub_account/xxx_image:latest` 為 Pod 模板中的 Image

```bash 
$ vi Dockerfile
```

```Dockerfile
FROM alpine:3.16
RUN apk --update add apache2
RUN rm -rf /var/cache/apk/*
ENTRYPOINT ["httpd", "-D", "FOREGROUND"]
```
- 記得自己打包然後丟上 Docker Hub

- 將 Deployment 名稱定義為 `apache-app-deployment`
- 維持 5 個 Running Pods
- 注意 labels 的使用

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
- 記得將 `your_dockerhub_account` 取代為你自己的 DockerHub 帳號
- Apache 應用程式監聽 80 port

**驗證：**
**預期結果：** 看見 5 個 Running Pods 使狀態成為 Ready "5/5"
```bash
$ kubectl get deployments apache-app-deployment
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
apache-app-deployment   5/5     5            5           20s
```

---

### 階段四：Service Apache

**要求：**
- 使用 NodePort 類別
- 將 Service 名稱定義為 `apache-app-service-nodeport`
- 將 Deployments Apache 中的 Pods 交由此 Service 管理網路請求
- 使用 8082 作為開放外界連線的 Port
- 注意 labels 的使用

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

**驗證：**
**預期結果：** 看見 NodePort 類別，Port 為 8082:3xxxx
```bash
$ kubectl get services apache-app-service-nodeport
NAME                          TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
apache-app-service-nodeport   NodePort   10.97.29.96   <none>        8082:30081/TCP   17s
```

```bash
$ curl [node_ip]:[node_exposed_port]
```
**預期結果：** 看見 Apache 首頁資訊
**提醒：** node_ip 指的是 minikube 所處在的 node ip，也就是所處在的 container ip

```bash
$ curl 192.168.49.2:30081
<html><body><h1>It works!</h1></body></html>
```

---

### 階段五：Ingress

**要求：**
- 先執行 `minikube addons enable ingress`，讓我們能去使用 nginx ingress controller
- 建立 rule，讓 `nginx.demo.com` 來源的請求送到你的 Service Nginx 處理
- 建立 rule，讓 `apache.demo.com` 來源的請求送到你的 Service Apache 處理

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

**驗證：**
**預期結果：** 稍等一陣子後，會看到 ingress ip、兩個 hostname、以及 80 port
```bash
$ kubectl get ingress -w
NAME               CLASS   HOSTS                            ADDRESS        PORTS   AGE
ingress-hostname   nginx   nginx.demo.com,apache.demo.com   192.168.49.2   80      7m50s
```

**預期結果：** 看見 Nginx 首頁資訊
```bash
$ curl 192.168.49.2:80 -H 'Host: nginx.demo.com'
```

**預期結果：** 看見 Apache 首頁資訊
```bash
$ curl 192.168.49.2:80 -H 'Host: apache.demo.com'
```

---

## 實作驗收

### 環境參數設定
全數完成後，設定以下環境參數：
```bash
NODE_IP=192.168.xxx.xxx
NODE_PORT_NGINX=3xxxx
NODE_PORT_APACHE=3xxxx
INGRESS_IP=192.168.xxx.xxx
```

### 驗收一：資源狀態
環境參數設定好後，執行以下指令並截圖：
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

### 驗收二：Nginx 服務測試
環境參數設定好後，執行以下指令並截圖：
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

### 驗收三：Apache 服務測試
環境參數設定好後，執行以下指令並截圖：
```bash
$ curl ${NODE_IP}:${NODE_PORT_APACHE}
<html><body><h1>It works!</h1></body></html>

$ curl ${INGRESS_IP}:80 -H 'Host: apache.demo.com'
<html><body><h1>It works!</h1></body></html>
```

### 驗收檔案清單
最後，yaml 檔案總計應有五個：

1. `nginx-deployment.yaml`
2. `nginx-service.yaml`
3. `apache-deployment.yaml`
4. `apache-service.yaml`
5. `hostname-ingress.yaml`
