# Install Kubernetes Dashboard Locally

[English](../en/42_install_k8s-dashboard_locally.md) | [繁體中文](../zh-tw/42_install_k8s-dashboard_locally.md) | [日本語](../ja/42_install_k8s-dashboard_locally.md) | [回到索引](../README.md)

- 先閱讀以下頁面：
  - [Kubernetes Dashboard](https://github.com/kubernetes/dashboard)

```shell
$ helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
"kubernetes-dashboard" has been added to your repositories

$ helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
Release "kubernetes-dashboard" does not exist. Installing it now.
NAME: kubernetes-dashboard
LAST DEPLOYED: Fri Sep 12 22:04:18 2025
NAMESPACE: kubernetes-dashboard
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
*************************************************************************************************
*** PLEASE BE PATIENT: Kubernetes Dashboard may need a few minutes to get up and become ready ***
*************************************************************************************************

Congratulations! You have just installed Kubernetes Dashboard in your cluster.

To access Dashboard run:
  kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443

NOTE: In case port-forward command does not work, make sure that kong service name is correct.
      Check the services in Kubernetes Dashboard namespace using:
        kubectl -n kubernetes-dashboard get svc

Dashboard will be available at:
  https://localhost:8443

$ kubectl -n kubernetes-dashboard get svc
NAME                                   TYPE        CLUSTER-IP        EXTERNAL-IP   PORT(S)    AGE
kubernetes-dashboard-api               ClusterIP   192.168.194.153   <none>        8000/TCP   101s
kubernetes-dashboard-auth              ClusterIP   192.168.194.233   <none>        8000/TCP   101s
kubernetes-dashboard-kong-proxy        ClusterIP   192.168.194.210   <none>        443/TCP    101s
kubernetes-dashboard-metrics-scraper   ClusterIP   192.168.194.171   <none>        8000/TCP   101s
kubernetes-dashboard-web               ClusterIP   192.168.194.176   <none>        8000/TCP   101s

# 以下 port forward 擇一使用即可：

# 在 foreground 跑的 port forward
$ kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443

# 在 background 跑的 port forward
$ nohup kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443 > port-forward.log 2>&1 &
```

- 連上 https://localhost:8443, 會發現要輸入 token
- 在電腦上建立以下檔案 (k8s-dashboard-admin.yaml)：
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```
- 這個是測試環境用的，正式環境不能這樣開權限，會有資安疑慮

- 執行以下指令：
```shell
$ kubectl apply -f ~/work/custom_scripts/k8s-dashboard-admin.yaml
serviceaccount/admin-user created

# 自己調整 token duration
$ kubectl -n kubernetes-dashboard create token admin-user --duration=1h
token generated here...
```

- 把上面的 token copy 下來就可以登入了 dashboard 了

- 可以再看一下現在 dashboard 啟動後，有哪些元件被建立：
```shell
$ kubectl -n kubernetes-dashboard get all
NAME                                                       READY   STATUS    RESTARTS   AGE
pod/kubernetes-dashboard-api-cc867f65c-r8rv8               1/1     Running   0          18m
pod/kubernetes-dashboard-auth-649fd6cfc4-t7ccf             1/1     Running   0          18m
pod/kubernetes-dashboard-kong-79867c9c48-mnhnj             1/1     Running   0          18m
pod/kubernetes-dashboard-metrics-scraper-547874fcf-rnzr2   1/1     Running   0          18m
pod/kubernetes-dashboard-web-7796b9fbbb-fvn55              1/1     Running   0          18m

NAME                                           TYPE        CLUSTER-IP        EXTERNAL-IP   PORT(S)    AGE
service/kubernetes-dashboard-api               ClusterIP   192.168.194.153   <none>        8000/TCP   18m
service/kubernetes-dashboard-auth              ClusterIP   192.168.194.233   <none>        8000/TCP   18m
service/kubernetes-dashboard-kong-proxy        ClusterIP   192.168.194.210   <none>        443/TCP    18m
service/kubernetes-dashboard-metrics-scraper   ClusterIP   192.168.194.171   <none>        8000/TCP   18m
service/kubernetes-dashboard-web               ClusterIP   192.168.194.176   <none>        8000/TCP   18m

NAME                                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/kubernetes-dashboard-api               1/1     1            1           18m
deployment.apps/kubernetes-dashboard-auth              1/1     1            1           18m
deployment.apps/kubernetes-dashboard-kong              1/1     1            1           18m
deployment.apps/kubernetes-dashboard-metrics-scraper   1/1     1            1           18m
deployment.apps/kubernetes-dashboard-web               1/1     1            1           18m

NAME                                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/kubernetes-dashboard-api-cc867f65c               1         1         1       18m
replicaset.apps/kubernetes-dashboard-auth-649fd6cfc4             1         1         1       18m
replicaset.apps/kubernetes-dashboard-kong-79867c9c48             1         1         1       18m
replicaset.apps/kubernetes-dashboard-metrics-scraper-547874fcf   1         1         1       18m
replicaset.apps/kubernetes-dashboard-web-7796b9fbbb              1         1         1       18m
```

- 要移除 dashboard 的話，一樣透過 helm:
```shell
$ helm uninstall kubernetes-dashboard -n kubernetes-dashboard
```
