# Experience Sharing: Installing Ingress Controller with Helm

[English](09_install_ingress_controller.md) | [繁體中文](../zh-tw/09_install_ingress_controller.md) | [日本語](../ja/09_install_ingress_controller.md) | [Back to Index](../README.md)

---

## Background
- Experiment Date: 2025/08/28
- Difficulty: 🤬
- Description: Originally encountered issues with outdated versions and complex configurations when using `kubectl apply -f deploy.yaml` to install ingress-nginx controller. Switching to Helm installation greatly simplified the process.

## Important Concept Clarification

In Kubernetes, there are two different concepts that need to be distinguished:

- **Ingress Controller**: The actual program that handles traffic (such as nginx), responsible for implementing Ingress rules
- **Ingress**: A Kubernetes resource that defines routing rules, telling the Controller how to forward traffic

Helm installs the **Ingress Controller**, and then we need to create **Ingress** resources separately to define routing rules.

---

## Issues Encountered

- Using the official `deploy.yaml` file to install ingress-nginx controller
- After installation, found the version was outdated, functionality incomplete, and Pods kept restarting
- Required manual handling of complex configurations like Webhook, Certificate, RBAC, etc.
- Each update required re-downloading new YAML files
- Removal required manually deleting multiple resources

## Troubleshooting Process

### Phase 1: Trying Traditional Installation Method

```bash
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.44.0/deploy/static/provider/cloud/deploy.yaml
```

**Issues encountered:**
- Fixed version, difficult to upgrade
- YAML file contained many manually configured resources
- Required manual handling of namespace, service account, cluster role, etc.
- Removal required manually deleting all related resources

### Phase 2: Switching to Helm Installation

Discovered that Helm could solve all the above problems, decided to switch to Helm installation.

## Solution

### Step 1: Install Ingress Controller with Helm

#### 1.1 Add Helm Repository
```bash
$ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```

#### 1.2 Update Repository
```bash
$ helm repo update
```

#### 1.3 Install Ingress Controller
```bash
$ helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

### Customized Installation (Optional)

If you need to use NodePort instead of the default LoadBalancer:

```bash
$ helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.http=30080 \
  --set controller.service.nodePorts.https=30443
```

### Step 2: Create Resources

After installing the Ingress Controller, we need to create Ingress resources to define routing rules:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-ingress
  annotations:
    # We are defining this annotation to prevent nginx
    # from redirecting requests to `https` for now
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: nginx # This line is necessary otherwise the ingress can not expose the address
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-svc
                port:
                  number: 1234
          - path: /hello
            pathType: Prefix
            backend:
              service:
                name: hellok8s-svc
                port:
                  number: 4567
```

Then the Services:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    app: nginx
  ports:
  - port: 1234
    targetPort: 80

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx-container

```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hellok8s-svc
spec:
  selector:
    app: hellok8s
  ports:
  - port: 4567
    targetPort: 4567

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: hellok8s
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hellok8s
  template:
    metadata:
      labels:
        app: hellok8s
    spec:
      containers:
      - image: brianstorti/hellok8s:v3
        name: hellok8s-container
```
Then apply in order.

## Verification

### Check Ingress Controller Status

#### Check Pod Status
```bash
$ kubectl get pods -n ingress-nginx
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-66cdbcf9bf-g995j   1/1     Running   0          44m
```

#### Check Service Status
```bash
$ kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP        EXTERNAL-IP     PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   192.168.194.148   192.168.139.2   80:31802/TCP,443:31809/TCP   44m
ingress-nginx-controller-admission   ClusterIP      192.168.194.149   <none>          443/TCP                      44m
```

### Check Ingress Resource Status
```bash
$ kubectl get ingress
NAME            CLASS   HOSTS   ADDRESS         PORTS   AGE
hello-ingress   nginx   *       192.168.139.2   80      42m
```

### Test Connection
```bash
curl http://192.168.139.2/hello
```

## Check Ingress Controller Version

### Method 1: Use Helm to Check Release Information
```bash
# Check detailed release information
$ helm list -n ingress-nginx
NAME         	NAMESPACE    	REVISION	UPDATED                             	STATUS  	CHART               	APP VERSION
ingress-nginx	ingress-nginx	1       	2025-08-28 21:13:57.830894 +0800 CST	deployed	ingress-nginx-4.13.1	1.13.1

# Check detailed release status
$ helm status ingress-nginx -n ingress-nginx
NAME: ingress-nginx
LAST DEPLOYED: Thu Aug 28 21:13:57 2025
NAMESPACE: ingress-nginx
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The ingress-nginx controller has been installed.
It may take a few minutes for the load balancer IP to be available.
You can watch the status by running 'kubectl get service --namespace ingress-nginx ingress-nginx-controller --output wide --watch'

An example Ingress that makes use of the controller:
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: example
    namespace: foo
  spec:
    ingressClassName: nginx
    rules:
      - host: www.example.com
        http:
          paths:
            - pathType: Prefix
              backend:
                service:
                  name: exampleService
                  port:
                    number: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
      - hosts:
        - www.example.com
        secretName: example-tls

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls
```

### Method 2: Check Chart Version Information
```bash
# Check chart metadata
$ helm get metadata ingress-nginx -n ingress-nginx
NAME: ingress-nginx
CHART: ingress-nginx
VERSION: 4.13.1
APP_VERSION: 1.13.1
ANNOTATIONS: artifacthub.io/changes=- 'Make: Add `helm-test` target. (#13659)'
- Update Ingress-Nginx version controller-v1.13.1
,artifacthub.io/prerelease=false
DEPENDENCIES:
NAMESPACE: ingress-nginx
REVISION: 1
STATUS: deployed
DEPLOYED_AT: 2025-08-28T21:13:57+08:00
```

### Method 3: Check Version Information in Pod
```bash
# Check detailed information of ingress controller pod
$ kubectl describe pod -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# Or directly check the pod's image tag
$ kubectl get pods -n ingress-nginx -o jsonpath='{.items[0].spec.containers[0].image}'
registry.k8s.io/ingress-nginx/controller:v1.13.1@sha256:37e489b22ac77576576e52e474941cd7754238438847c1ee795ad6d59c02b12a%
```

### Method 4: Check Available Chart Versions
```bash
# Check available versions in remote repository
$ helm search repo ingress-nginx/ingress-nginx --versions
NAME                       	CHART VERSION	APP VERSION	DESCRIPTION
ingress-nginx/ingress-nginx	4.13.1       	1.13.1     	Ingress controller for Kubernetes using NGINX a...
ingress-nginx/ingress-nginx	4.13.0       	1.13.0     	Ingress controller for Kubernetes using NGINX a...
ingress-nginx/ingress-nginx	4.12.5       	1.12.5     	Ingress controller for Kubernetes using NGINX a...
ingress-nginx/ingress-nginx	4.12.4       	1.12.4     	Ingress controller for Kubernetes using NGINX a...
...

# Check local chart information
$ helm show chart ingress-nginx/ingress-nginx
```

## Why is Helm Better?

| Item | kubectl apply | Helm |
|------|---------------|------|
| Version Management | Need to manually download new versions | Can specify version, automatic handling |
| Configuration Complexity | Need to manually handle all resources | Automatically handles Webhook, Certificate, RBAC |
| Customization | Need to modify YAML | Use `--set` parameters |
| Removal | Need to manually delete multiple resources | `helm uninstall` completes in one step |
| Upgrade | Need to re-download and apply | `helm upgrade` handles automatically |

## Summary

The original method using `kubectl apply -f deploy.yaml` was direct but encountered issues with version management and configuration complexity. After switching to Helm, the entire installation process became more concise and easier to maintain, especially suitable for environments that require frequent updates or customization.

## Important Reminder

**Helm installs the Ingress Controller, not the Ingress itself**. After installing the Controller, you still need to:

1. Create Ingress resources to define routing rules
2. Ensure the `ingressClassName` in Ingress resources matches the Controller
3. Create corresponding Services and Pods to handle actual business logic

This architecture separates Ingress Controller and Ingress rules, providing better flexibility and maintainability.
