# Kubernetes Practice Exercise

[English](../en/34_k8s_practice.md) | [繁體中文](../zh-tw/34_k8s_practice.md) | [日本語](../ja/34_k8s_practice.md) | [Back to Index](../README.md)

## Preliminary Setup

### 1. Clean Existing Environment
```bash
$ minikube stop
$ minikube delete
$ minikube status
```

### 2. Create Fresh Minikube Cluster
```bash
$ minikube start --driver docker
```

---

## Template Design Phase

### Phase 1: Nginx Deployment

**Requirements:**
- Use `nginx:1.29` (as of 2025/07) as the Image name in Pod template
- Define Deployment name as `nginx-app-deployment`
- Maintain 5 Running Pods
- Pay attention to labels usage

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

**Verification:**
**Expected Result:** See 5 Running Pods with Ready status "5/5"
```bash
$ kubectl get deployments nginx-app-deployment
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
nginx-app-deployment   5/5     5            5           19s
```

---

### Phase 2: Nginx Service

**Requirements:**
- Use NodePort type
- Define Service name as `nginx-app-service-nodeport`
- Manage network requests for Pods in Nginx Deployment through this Service
- Use 8081 as the port exposed to external connections
- Pay attention to labels usage

**Notes:**
- Nginx application listens on port 80

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

**Verification:**
**Expected Result:** See NodePort type, Port as 8081:3xxxx
```bash
$ kubectl get services nginx-app-service-nodeport
NAME                         TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
nginx-app-service-nodeport   NodePort   10.107.180.108   <none>        8081:30080/TCP   27s
```

**Expected Result:** See Nginx homepage information (ex. `<h1>Welcome to nginx!</h1>`)
**Reminder:** node_ip refers to the node IP where minikube is located, which is the container IP
```bash
Get node ip
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

### Phase 3: Apache Deployment

**Requirements:**
- Use the custom-built `your_dockerhub_account/xxx_image:latest` from Docker exercises as the Image in Pod template

```bash 
$ vi Dockerfile
```

```Dockerfile
FROM alpine:3.16
RUN apk --update add apache2
RUN rm -rf /var/cache/apk/*
ENTRYPOINT ["httpd", "-D", "FOREGROUND"]
```
- Remember to build and push to Docker Hub yourself

- Define Deployment name as `apache-app-deployment`
- Maintain 5 Running Pods
- Pay attention to labels usage

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

**Notes:**
- Remember to replace `your_dockerhub_account` with your own DockerHub account
- Apache application listens on port 80

**Verification:**
**Expected Result:** See 5 Running Pods with Ready status "5/5"
```bash
$ kubectl get deployments apache-app-deployment
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
apache-app-deployment   5/5     5            5           20s
```

---

### Phase 4: Apache Service

**Requirements:**
- Use NodePort type
- Define Service name as `apache-app-service-nodeport`
- Manage network requests for Pods in Apache Deployment through this Service
- Use 8082 as the port exposed to external connections
- Pay attention to labels usage

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

**Verification:**
**Expected Result:** See NodePort type, Port as 8082:3xxxx
```bash
$ kubectl get services apache-app-service-nodeport
NAME                          TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
apache-app-service-nodeport   NodePort   10.97.29.96   <none>        8082:30081/TCP   17s
```

```bash
$ curl [node_ip]:[node_exposed_port]
```
**Expected Result:** See Apache homepage information
**Reminder:** node_ip refers to the node IP where minikube is located, which is the container IP

```bash
$ curl 192.168.49.2:30081
<html><body><h1>It works!</h1></body></html>
```

---

### Phase 5: Ingress

**Requirements:**
- First run `minikube addons enable ingress` to enable nginx ingress controller
- Create rule to route requests from `nginx.demo.com` to your Nginx Service
- Create rule to route requests from `apache.demo.com` to your Apache Service

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

**Verification:**
**Expected Result:** After waiting for a while, you'll see ingress IP, two hostnames, and port 80
```bash
$ kubectl get ingress -w
NAME               CLASS   HOSTS                            ADDRESS        PORTS   AGE
ingress-hostname   nginx   nginx.demo.com,apache.demo.com   192.168.49.2   80      7m50s
```

**Expected Result:** See Nginx homepage information
```bash
$ curl 192.168.49.2:80 -H 'Host: nginx.demo.com'
```

**Expected Result:** See Apache homepage information
```bash
$ curl 192.168.49.2:80 -H 'Host: apache.demo.com'
```

---

## Implementation Verification

### Environment Parameter Setup
After completing everything, set the following environment parameters:
```bash
NODE_IP=192.168.xxx.xxx
NODE_PORT_NGINX=3xxxx
NODE_PORT_APACHE=3xxxx
INGRESS_IP=192.168.xxx.xxx
```

### Verification 1: Resource Status
After setting environment parameters, execute the following commands and take screenshots:
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

### Verification 2: Nginx Service Testing
After setting environment parameters, execute the following commands and take screenshots:
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

### Verification 3: Apache Service Testing
After setting environment parameters, execute the following commands and take screenshots:
```bash
$ curl ${NODE_IP}:${NODE_PORT_APACHE}
<html><body><h1>It works!</h1></body></html>

$ curl ${INGRESS_IP}:80 -H 'Host: apache.demo.com'
<html><body><h1>It works!</h1></body></html>
```

### Verification File List
Finally, there should be a total of five yaml files:

1. `nginx-deployment.yaml`
2. `nginx-service.yaml`
3. `apache-deployment.yaml`
4. `apache-service.yaml`
5. `hostname-ingress.yaml`