# AWS EKS 實作指南

[English](../en/35_eks_practice.md) | [繁體中文](../zh-tw/35_eks_practice.md) | [日本語](../ja/35_eks_practice.md) | [回到索引](../README.md)

## 目錄
1. [環境準備](#環境準備)
2. [EKS 叢集建立](#eks-叢集建立)
3. [EFS 持久化儲存設定](#efs-持久化儲存設定)
4. [Load Balancer Controller 設定](#load-balancer-controller-設定)
5. [多階段部署專案](#多階段部署專案)
6. [HPA 自動擴展設定](#hpa-自動擴展設定)
7. [資源清理](#資源清理)

---

## 環境準備

### 前置需求
在開始 EKS 實作前，需要安裝以下工具：

#### kubectl 安裝
```bash
$ curl -o kubectl https://s3.us-west-2.amazonaws.com/amazon-eks/1.33.0/2025-05-01/bin/linux/amd64/kubectl
$ chmod +x ./kubectl
$ mkdir -p $HOME/bin
$ cp ./kubectl $HOME/bin/kubectl
$ export PATH=$PATH:$HOME/bin
$ echo 'export PATH=$PATH:$HOME/bin' >> ~/.zshrc
$ kubectl version --short --client
```

#### eksctl 安裝
```bash
$ mkdir ~/tmp
$ curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C ~/tmp
$ mv ~/tmp/eksctl $HOME/bin/eksctl
$ eksctl version
```

#### Helm 安裝
```bash
$ curl -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
$ helm version --short
```

#### Git 安裝
```bash
$ sudo yum install -y git
```

#### AWS CLI 安裝
```bash
$ aws --version
$ sudo rm -rf /usr/bin/aws
$ curl -o "awscliv2.zip" "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip"
$ unzip awscliv2.zip
$ sudo ./aws/install
$ aws --version
```

### AWS 權限設定
```bash
$ aws configure
```

### 環境變數設定
```bash
$ CLUSTER_NAME=your_cluster_name
$ echo ${CLUSTER_NAME}

$ AWS_REGION=your_aws_region
$ echo ${AWS_REGION}

$ AWS_ACCOUNT=your_aws_account
$ echo ${AWS_ACCOUNT}
```

- 以上設定都已經封裝在 [ssm_user_setup.sh](https://github.com/nekowanderer/nebuletta/blob/main/scripts/ssm_user_setup.sh) 腳本之中，可調整 `Configurations` 區塊的變數後直接在 EC2 上執行，相關說明可參考此[連結](https://github.com/nekowanderer/nebuletta/blob/main/scripts/ssm_user_setup.sh)。
- 完成後，也可以直接將家目錄下的 `.zshrc` 替換成[此處的內容方便接下來的練習](https://github.com/nekowanderer/nebuletta/blob/main/scripts/custom_zshrc)。

---

## EKS Cluster 建立
此步驟的細部設定請參考 [EKS 建立設定指南](./36_eks_cluster_setup.md)

### 建立 Fargate 叢集
```bash
$ eksctl create cluster --name ${CLUSTER_NAME} --version 1.33 --fargate
2025-07-06 05:41:10 [ℹ]  eksctl version 0.210.0
2025-07-06 05:41:10 [ℹ]  using region ap-northeast-1
2025-07-06 05:41:11 [ℹ]  setting availability zones to [ap-northeast-1d ap-northeast-1c ap-northeast-1a]
2025-07-06 05:41:11 [ℹ]  subnets for ap-northeast-1d - public:192.168.0.0/19 private:192.168.96.0/19
2025-07-06 05:41:11 [ℹ]  subnets for ap-northeast-1c - public:192.168.32.0/19 private:192.168.128.0/19
2025-07-06 05:41:11 [ℹ]  subnets for ap-northeast-1a - public:192.168.64.0/19 private:192.168.160.0/19
2025-07-06 05:41:11 [ℹ]  using Kubernetes version 1.33
2025-07-06 05:41:11 [ℹ]  creating EKS cluster "my-cluster-001" in "ap-northeast-1" region with Fargate profile
2025-07-06 05:41:11 [ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=ap-northeast-1 --cluster=my-cluster-001'
2025-07-06 05:41:11 [ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "my-cluster-001" in "ap-northeast-1"
2025-07-06 05:41:11 [ℹ]  CloudWatch logging will not be enabled for cluster "my-cluster-001" in "ap-northeast-1"
2025-07-06 05:41:11 [ℹ]  you can enable it with 'eksctl utils update-cluster-logging --enable-types={SPECIFY-YOUR-LOG-TYPES-HERE (e.g. all)} --region=ap-northeast-1 --cluster=my-cluster-001'
2025-07-06 05:41:11 [ℹ]  default addons vpc-cni, kube-proxy, coredns, metrics-server were not specified, will install them as EKS addons
2025-07-06 05:41:11 [ℹ]
2 sequential tasks: { create cluster control plane "my-cluster-001",
    3 sequential sub-tasks: {
        1 task: { create addons },
        wait for control plane to become ready,
        create fargate profiles,
    }
}
2025-07-06 05:41:11 [ℹ]  building cluster stack "eksctl-my-cluster-001-cluster"
2025-07-06 05:41:11 [ℹ]  deploying stack "eksctl-my-cluster-001-cluster"
2025-07-06 05:41:41 [ℹ]  waiting for CloudFormation stack "eksctl-my-cluster-001-cluster"
2025-07-06 05:42:11 [ℹ]  waiting for CloudFormation stack "eksctl-my-cluster-001-cluster"
2025-07-06 05:43:11 [ℹ]  waiting for CloudFormation stack "eksctl-my-cluster-001-cluster"
2025-07-06 05:44:11 [ℹ]  waiting for CloudFormation stack "eksctl-my-cluster-001-cluster"
2025-07-06 05:45:11 [ℹ]  waiting for CloudFormation stack "eksctl-my-cluster-001-cluster"
2025-07-06 05:46:11 [ℹ]  waiting for CloudFormation stack "eksctl-my-cluster-001-cluster"
2025-07-06 05:47:11 [ℹ]  waiting for CloudFormation stack "eksctl-my-cluster-001-cluster"
2025-07-06 05:48:11 [ℹ]  waiting for CloudFormation stack "eksctl-my-cluster-001-cluster"
2025-07-06 05:49:11 [ℹ]  waiting for CloudFormation stack "eksctl-my-cluster-001-cluster"
2025-07-06 05:49:12 [!]  recommended policies were found for "vpc-cni" addon, but since OIDC is disabled on the cluster, eksctl cannot configure the requested permissions; the recommended way to provide IAM permissions for "vpc-cni" addon is via pod identity associations; after addon creation is completed, add all recommended policies to the config file, under `addon.PodIdentityAssociations`, and run `eksctl update addon`
2025-07-06 05:49:12 [ℹ]  creating addon: vpc-cni
2025-07-06 05:49:13 [ℹ]  successfully created addon: vpc-cni
2025-07-06 05:49:13 [ℹ]  creating addon: kube-proxy
2025-07-06 05:49:13 [ℹ]  successfully created addon: kube-proxy
2025-07-06 05:49:14 [ℹ]  creating addon: coredns
2025-07-06 05:49:14 [ℹ]  successfully created addon: coredns
2025-07-06 05:49:15 [ℹ]  creating addon: metrics-server
2025-07-06 05:49:15 [ℹ]  successfully created addon: metrics-server
2025-07-06 05:51:15 [ℹ]  creating Fargate profile "fp-default" on EKS cluster "my-cluster-001"
2025-07-06 05:53:26 [ℹ]  created Fargate profile "fp-default" on EKS cluster "my-cluster-001"
2025-07-06 05:53:56 [ℹ]  "coredns" is now schedulable onto Fargate
2025-07-06 05:54:59 [ℹ]  "coredns" is now scheduled onto Fargate
2025-07-06 05:54:59 [ℹ]  "coredns" pods are now scheduled onto Fargate
2025-07-06 05:54:59 [ℹ]  waiting for the control plane to become ready
2025-07-06 05:55:00 [✔]  saved kubeconfig as "/home/ssm-user/.kube/config"
2025-07-06 05:55:00 [ℹ]  no tasks
2025-07-06 05:55:00 [✔]  all EKS cluster resources for "my-cluster-001" have been created
2025-07-06 05:55:03 [ℹ]  kubectl command should work with "/home/ssm-user/.kube/config", try 'kubectl get nodes'
2025-07-06 05:55:03 [✔]  EKS cluster "my-cluster-001" in "ap-northeast-1" region is ready
```

### 檢查安裝結果
```bash
$ cat .kube/config
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURCVENDQWUyZ0F3SUJBZ0lJQjZKUVNvblViREl3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TlRBM01EWXdOVFF4TVRCYUZ3MHpOVEEzTURRd05UUTJNVEJhTUJVeApFekFSQmdOVkJBTVRDbXQxWW1WeWJtVjBaWE13Z2dFaU1BMEdDU3FHU0liM0RRRUJBUVVBQTRJQkR3QXdnZ0VLCkFvSUJBUURKKzRrZ2YxMHlXVVZINmw0Wmd6Z3NWc3YyV3VKTUNnQ1oyb3YvSjRoNXlmeUVqamlyRXBKdEVZUzQKbW81ZUR1Y0VjRCtqRHEwNDltQ1BNUGlmcHFQZ3JGU2tmMlNtQktFM1B6Y2ZkUWRxRmx3d0VhUnhBTzQ5WEx2eAp1Yk9kYTRFTGdMNnIzNm82YlFXL0FoQXNUYzV2NGJzY2NoeWkvYm5XY0dXNGpJZUJXd2dxb1J4SEpLQlRqN0p5Cm1LV0lFUEdpdXNsUG8yWTlLdEJudlFvMTJxd3RKT01Ma3V1ZUlmR0E1R09PbnVuTy9DS09Ick5LTk1INkpldUgKcXQ2M09YZ3lUVWUvclVQMzJHbTZqZUdnR1RISWpVS3NObHRLVk4zVzlFVjdPa3dVME83MkxpS05NdkdJM2x5QQpUVWN4bWN0WmhCYi9nUU5NYWh6dGRicHBtbDVCQWdNQkFBR2pXVEJYTUE0R0ExVWREd0VCL3dRRUF3SUNwREFQCkJnTlZIUk1CQWY4RUJUQURBUUgvTUIwR0ExVWREZ1FXQkJUejFZQlRqc1ZjZkFsYnk5TzBGWEh1OElXdEh6QVYKQmdOVkhSRUVEakFNZ2dwcmRXSmxjbTVsZEdWek1BMEdDU3FHU0liM0RRRUJDd1VBQTRJQkFRQTFtTWcyNktkUwpiUFdXbXMxdzl6WXdWdFYvSkF3OUFIaTRTaklRYUFOS2FLQUVoWi9ReEFUdWJ3bEZkOWtUK085bk11VmZzb0pLCmJrRTJyc25wZG1tYkxoQnhHeTVoVHUvMlE0Q0JOZHFuWldsRlo5bVNGNUFtZTh3WTNVcHkySmlsZ1JFMkM4eXgKM0F1MCttZU84RDFKSUpESGczZ25BbXBNemFsSVlnSXFHL215QjloZk1vVWZXR3pHRm56UW9mZ3d3OEs3cTkwNgpRUXVHWkxLZThldVJ5S1Y4VWVKUlF4MXhadzlEN1h0OEFycXplcys0dlpiRy96b2Q0ZXJCUGMxQmdLNGp3cXBhCkdpRVFHS2Z6L3B1N0dVWW1GRHZlRTczVTVSNDQ2Y0ROa29QL1QxRGh5N1hnOWpveUJWcVRDaHl4b3BBdk9xc0MKeXgwRENzcTgvYzVOCi0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    server: https://8B8DC61011BE24084D76FEBB4DDDD9C3.sk1.ap-northeast-1.eks.amazonaws.com
  name: my-cluster-001.ap-northeast-1.eksctl.io
contexts:
- context:
    cluster: my-cluster-001.ap-northeast-1.eksctl.io
    user: i-0a4cd97300f5a0107@my-cluster-001.ap-northeast-1.eksctl.io
  name: i-0a4cd97300f5a0107@my-cluster-001.ap-northeast-1.eksctl.io
current-context: i-0a4cd97300f5a0107@my-cluster-001.ap-northeast-1.eksctl.io
kind: Config
preferences: {}
users:
- name: i-0a4cd97300f5a0107@my-cluster-001.ap-northeast-1.eksctl.io
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      args:
      - eks
      - get-token
      - --output
      - json
      - --cluster-name
      - my-cluster-001
      - --region
      - ap-northeast-1
      command: aws
      env:
      - name: AWS_STS_REGIONAL_ENDPOINTS
        value: regional
      provideClusterInfo: false
```

### 確認 kube-system namespace 下的各個元件
```bash
$ kubectl get all -n kube-system
NAME                                  READY   STATUS    RESTARTS   AGE
pod/coredns-5ff77f65d4-jn6d9          1/1     Running   0          2m36s
pod/coredns-5ff77f65d4-pqwtk          1/1     Running   0          2m36s
pod/metrics-server-7f76d4758d-db2vs   0/1     Pending   0          5m41s
pod/metrics-server-7f76d4758d-sst8j   0/1     Pending   0          5m41s

NAME                                TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                  AGE
service/eks-extension-metrics-api   ClusterIP   10.100.83.197    <none>        443/TCP                  9m44s
service/kube-dns                    ClusterIP   10.100.0.10      <none>        53/UDP,53/TCP,9153/TCP   6m12s
service/metrics-server              ClusterIP   10.100.122.161   <none>        443/TCP                  5m41s

NAME                        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/aws-node     0         0         0       0            0           <none>          5m43s
daemonset.apps/kube-proxy   0         0         0       0            0           <none>          5m43s

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coredns          2/2     2            2           6m12s
deployment.apps/metrics-server   0/2     2            0           5m41s

NAME                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/coredns-5c658475b5          0         0         0       6m12s
replicaset.apps/coredns-5ff77f65d4          2         2         2       2m36s
replicaset.apps/metrics-server-7f76d4758d   2         2         0       5m41s
```
- 這裡最重要的是 coredns，這個元件會協助我們進行 domain name 的相關處理。

### 建立 OIDC Provider
```bash
$ eksctl utils associate-iam-oidc-provider --cluster ${CLUSTER_NAME} --approve
2025-07-06 05:56:55 [ℹ]  will create IAM Open ID Connect provider for cluster "my-cluster-001" in "ap-northeast-1"
2025-07-06 05:56:56 [✔]  created IAM Open ID Connect provider for cluster "my-cluster-001" in "ap-northeast-1"
```

### 刪除 EKS Cluster 
後面還有很多相關練習，怕浪費錢的話就先砍掉，要用的時候再建立：
```bash
$ eksctl delete cluster --region=ap-northeast-1 --name=my-cluster-001
2025-07-06 06:00:02 [ℹ]  deleting EKS cluster "my-cluster-001"
2025-07-06 06:00:03 [ℹ]  deleting Fargate profile "fp-default"
2025-07-06 06:02:11 [ℹ]  deleted Fargate profile "fp-default"
2025-07-06 06:02:11 [ℹ]  deleted 1 Fargate profile(s)
2025-07-06 06:02:11 [✔]  kubeconfig has been updated
2025-07-06 06:02:11 [ℹ]  cleaning up AWS load balancers created by Kubernetes objects of Kind Service or Ingress
2025-07-06 06:02:12 [ℹ]
2 sequential tasks: { delete IAM OIDC provider, delete cluster control plane "my-cluster-001" [async]
}
2025-07-06 06:02:13 [ℹ]  will delete stack "eksctl-my-cluster-001-cluster"
2025-07-06 06:02:13 [✔]  all cluster resources were deleted
```

---

## EFS 持久化儲存設定

### 建立 Security Group
<img src="../images/35_create_eks_efs_sg.jpg" width=600 />

- 名稱：`eks-efs-sg`
- 選擇 EKS 叢集的 VPC
- 允許全部連線

### 建立 EFS 檔案系統
<img src="../images/35_create_efs_1.jpg" width=600 />
<img src="../images/35_create_efs_2.jpg" width=600 />
<img src="../images/35_create_efs_3.jpg" width=600 />

- 名稱：`eks-efs`
- 選擇 EKS 叢集的 VPC
- 更新掛載點，使用 Security Group：`eks-efs-sg`
- 等待掛載點可用

### 安裝 EFS CSI Driver
```bash
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/deploy/kubernetes/base/csidriver.yaml
csidriver.storage.k8s.io/efs.csi.aws.com configured

$ kubectl get csidriver
NAME              ATTACHREQUIRED   PODINFOONMOUNT   STORAGECAPACITY   TOKENREQUESTS   REQUIRESREPUBLISH   MODES        AGE
efs.csi.aws.com   false            false            false             <unset>         false               Persistent   24m
```

### 建立 Persistent Volume (PV)
```bash
$ git clone https://github.com/uopsdod/k8sOnCloud_hiskio.git
$ cd ~/k8sOnCloud_hiskio/aws_eks/initial
$ cp simple-volume-pv.yaml aws-efs-volume-pv.yaml
```

編輯 `aws-efs-volume-pv.yaml`：
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: app-pv
spec:
  storageClassName: sc-001
  volumeMode: Filesystem
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-0a1700a34bf9e8d24  # 替換成你剛才建立的 EFS 檔案系統 ID
```

### 部署 PV 和 PVC
```bash
$ kubectl apply -f aws-efs-volume-pv.yaml
persistentvolume/app-pv created

$ kubectl get pv
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
app-pv   2Gi        RWX            Retain           Available           sc-001         <unset>                          16s

$ kubectl describe pv app-pv
Name:            app-pv
Labels:          <none>
Annotations:     <none>
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:    sc-001
Status:          Available
Claim:
Reclaim Policy:  Retain
Access Modes:    RWX
VolumeMode:      Filesystem
Capacity:        2Gi
Node Affinity:   <none>
Message:
Source:
    Type:              CSI (a Container Storage Interface (CSI) volume source)
    Driver:            efs.csi.aws.com
    FSType:
    VolumeHandle:      fs-0a1700a34bf9e8d24
    ReadOnly:          false
    VolumeAttributes:  <none>
Events:                <none>  # 重點是這行，注意有沒有 error message
```


編輯 `simple-volume-pvc.yaml`：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc
spec:
  storageClassName: sc-001
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
```

```bash
$ kubectl apply -f simple-volume-pvc.yaml
persistentvolumeclaim/app-pvc created

$ kubectl get pvc
NAME      STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
app-pvc   Bound    app-pv   2Gi        RWX            sc-001         <unset>                 29s

$ kubectl describe pvc app-pvc
Name:          app-pvc
Namespace:     default
StorageClass:  sc-001
Status:        Bound
Volume:        app-pv
Labels:        <none>
Annotations:   pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      2Gi
Access Modes:  RWX
VolumeMode:    Filesystem
Used By:       <none>
Events:        <none> # 重點是這行，注意有沒有 error message
```

### 部署應用程式
編輯 `simple-deployment-volume.yaml`：

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
        ports:
        - containerPort: 80
        volumeMounts:
          - name: app-volume
            mountPath: /app/data
      volumes:
        - name: app-volume
          persistentVolumeClaim:
            claimName: app-pvc
```

```bash
$ kubectl apply -f simple-deployment-volume.yaml
deployment.apps/app-deployment created

$ kubectl get deployments
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
app-deployment   0/3     3            0           22s

$ kubectl get pods -w
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
app-deployment   0/3     3            0           25s
app-deployment   1/3     3            1           77s
app-deployment   2/3     3            2           80s
app-deployment   3/3     3            3           84s
```

### 測試持久化儲存
```bash
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
app-deployment-58449898c5-4fqws   1/1     Running   0          2m31s
app-deployment-58449898c5-7m54s   1/1     Running   0          2m31s
app-deployment-58449898c5-hbblt   1/1     Running   0          2m31s

$ kubectl exec -it [pod_name] -- touch /app/data/file003.txt
$ kubectl exec -it [pod_name] -- ls /app/data
file001.txt

$ kubectl delete pods --all
pod "app-deployment-58449898c5-4fqws" deleted
pod "app-deployment-58449898c5-7m54s" deleted
pod "app-deployment-58449898c5-hbblt" deleted

$ kubectl get pods -w
NAME                              READY   STATUS              RESTARTS   AGE
app-deployment-58449898c5-4jgxk   0/1     ContainerCreating   0          57s
app-deployment-58449898c5-mcsjj   0/1     ContainerCreating   0          57s
app-deployment-58449898c5-njmp8   0/1     ContainerCreating   0          57s
app-deployment-58449898c5-njmp8   1/1     Running             0          68s
app-deployment-58449898c5-mcsjj   1/1     Running             0          69s
app-deployment-58449898c5-4jgxk   1/1     Running             0          72s

$ kubectl exec -it [pod_name] -- ls /app/data
file001.txt
```

---

## Load Balancer Controller 設定

### 建立 IAM Policy
```bash
$ curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

$ aws iam create-policy \
   --policy-name AWSLoadBalancerControllerIAMPolicy \
   --policy-document file://iam_policy.json

$ aws iam get-policy --policy-arn arn:aws:iam::${AWS_ACCOUNT}:policy/AWSLoadBalancerControllerIAMPolicy
{
  "Policy": {
    "PolicyName": "AWSLoadBalancerControllerIAMPolicy",
    "PolicyId": "ANPAVIYDROPBYUTZQRYZQ",
    "Arn": "arn:aws:iam::362395300803:policy/AWSLoadBalancerControllerIAMPolicy",
    "Path": "/",
    "DefaultVersionId": "v1",
    "AttachmentCount": 0,
    "PermissionsBoundaryUsageCount": 0,
    "IsAttachable": true,
    "CreateDate": "2025-07-11T13:42:41+00:00",
    "UpdateDate": "2025-07-11T13:42:41+00:00",
    "Tags": []
  }
}
```
- 有了這個 IAM policy 之後，就可以去建立一個 k8s 的 service account

### 建立 Service Account
```bash
$ eksctl create iamserviceaccount \
  --cluster=${CLUSTER_NAME} \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::${AWS_ACCOUNT}:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve
2025-07-11 13:47:34 [ℹ]  1 iamserviceaccount (kube-system/aws-load-balancer-controller) was included (based on the include/exclude rules)
2025-07-11 13:47:34 [!]  metadata of serviceaccounts that exist in Kubernetes will be updated, as --override-existing-serviceaccounts was set
2025-07-11 13:47:34 [ℹ]  1 task: {
    2 sequential sub-tasks: {
        create IAM role for serviceaccount "kube-system/aws-load-balancer-controller",
        create serviceaccount "kube-system/aws-load-balancer-controller",
    } }2025-07-11 13:47:34 [ℹ]  building iamserviceaccount stack "eksctl-my-cluster-001-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
2025-07-11 13:47:34 [ℹ]  deploying stack "eksctl-my-cluster-001-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
2025-07-11 13:47:34 [ℹ]  waiting for CloudFormation stack "eksctl-my-cluster-001-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
2025-07-11 13:48:04 [ℹ]  waiting for CloudFormation stack "eksctl-my-cluster-001-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
2025-07-11 13:48:04 [ℹ]  created serviceaccount "kube-system/aws-load-balancer-controller"
```
- Service account 可以讓 load balancer controller 具有需要的相關權限

```bash
$ eksctl get iamserviceaccount --cluster ${CLUSTER_NAME} --name aws-load-balancer-controller --namespace kube-system
NAMESPACE       NAME                            ROLE ARN
kube-system     aws-load-balancer-controller    arn:aws:iam::362395300803:role/eksctl-my-cluster-001-addon-iamserviceaccount-Role1-R6G3upoEDBOc
```
- 這邊有看到輸出就表示 service account 建立成功了

### 安裝 Load Balancer Controller
```bash
$ helm repo add eks https://aws.github.io/eks-charts
"eks" has been added to your repositories

$ curl -O https://raw.githubusercontent.com/aws/eks-charts/master/stable/aws-load-balancer-controller/crds/crds.yaml
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 32484  100 32484    0     0   141k      0 --:--:-- --:--:-- --:--:--  141k

$ kubectl apply -f crds.yaml
customresourcedefinition.apiextensions.k8s.io/ingressclassparams.elbv2.k8s.aws created
customresourcedefinition.apiextensions.k8s.io/targetgroupbindings.elbv2.k8s.aws created

# 查詢一下 EKS cluster 所使用的 VPC ID
$ aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.resourcesVpcConfig.vpcId" --output json | jq
"vpc-0cf3a9effdea78521"

$ VPC_ID=vpc-0cf3a9effdea78521  # 替換為你在上面查到的 VPC ID

$ echo ${VPC_ID}
```

- 東西都齊全了，可以建立 load balancer controller 了：

```bash
$ helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
    --set clusterName=${CLUSTER_NAME} \
    --set serviceAccount.create=false \
    --set region=${AWS_REGION} \
    --set vpcId=${VPC_ID} \
    --set serviceAccount.name=aws-load-balancer-controller \
    -n kube-system
NAME: aws-load-balancer-controller
LAST DEPLOYED: Fri Jul 11 14:01:09 2025
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
AWS Load Balancer controller installed!
```

- 觀察一下 aws-load-balancer 的狀態，等待到 ready 為止：

```bash
$ kubectl get all -n kube-system
NAME                                                READY   STATUS    RESTARTS   AGE
pod/aws-load-balancer-controller-5b895c4dd6-7g2cm   1/1     Running   0          90s
pod/aws-load-balancer-controller-5b895c4dd6-lt7r4   1/1     Running   0          90s
pod/coredns-5ff77f65d4-4zv6x                        1/1     Running   0          38m
pod/coredns-5ff77f65d4-w9tjw                        1/1     Running   0          38m
pod/metrics-server-6f674c86fd-859m7                 0/1     Pending   0          41m
pod/metrics-server-6f674c86fd-qv6d4                 0/1     Pending   0          41m

NAME                                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                  AGE
service/aws-load-balancer-webhook-service   ClusterIP   10.100.176.143   <none>        443/TCP                  90s
service/eks-extension-metrics-api           ClusterIP   10.100.34.26     <none>        443/TCP                  45m
service/kube-dns                            ClusterIP   10.100.0.10      <none>        53/UDP,53/TCP,9153/TCP   41m
service/metrics-server                      ClusterIP   10.100.230.212   <none>        443/TCP                  41m

NAME                        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/aws-node     0         0         0       0            0           <none>          41m
daemonset.apps/kube-proxy   0         0         0       0            0           <none>          41m

NAME                                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/aws-load-balancer-controller   2/2     2            2           90s
deployment.apps/coredns                        2/2     2            2           41m
deployment.apps/metrics-server                 0/2     2            0           41m

NAME                                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/aws-load-balancer-controller-5b895c4dd6   2         2         2       90s
replicaset.apps/coredns-5c658475b5                        0         0         0       41m
replicaset.apps/coredns-5ff77f65d4                        2         2         2       38m
replicaset.apps/metrics-server-6f674c86fd                 2         2         0       41m
```

---

## 多階段部署專案

### 建立 Fargate Profile
```bash
$ eksctl create fargateprofile --cluster ${CLUSTER_NAME} --region ${AWS_REGION} --name app-fp --namespace app-ns
2025-07-11 14:05:22 [ℹ]  creating Fargate profile "app-fp" on EKS cluster "my-cluster-001"
2025-07-11 14:05:40 [ℹ]  created Fargate profile "app-fp" on EKS cluster "my-cluster-001"
```
- fargateprofile 是 AWS EKS 上的特有概念，可以想成是統整運算資源以及各種配置，以配合 AWS 進行 K8S 的部署
- 此處的 namespace 跟之後要部署在其中的專案必須要是相同的

### 部署應用程式

- 建立 beta-app-all-hpa.yaml:
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
  replicas: 1
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
        resources:
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
  type: ClusterIP
  selector:
    app: beta-app-pod
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
---
```

- 建立 prod-app-all.yaml:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app-ns
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prod-app-deployment
  namespace: app-ns
spec:
  replicas: 3
  selector:
    matchLabels:
      app: prod-app-pod
  template:
    metadata:
      labels:
        app: prod-app-pod
    spec:
      containers:
      - name: prod-app-container
        image: uopsdod/k8s-hostname-amd64-prod:v1
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: prod-app-service-clusterip
  namespace: app-ns
spec:
  type: ClusterIP
  selector:
    app: prod-app-pod
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 80
---
```

- 進行 apply 然後檢查結果：

```bash
$ kubectl apply -f beta-app-all-hpa.yaml
namespace/app-ns created
deployment.apps/beta-app-deployment created
service/beta-app-service-clusterip created

$ kubectl apply -f prod-app-all.yaml
namespace/app-ns unchanged
deployment.apps/prod-app-deployment created
service/prod-app-service-clusterip created

$ kubectl get all -n app-ns
NAME                                       READY   STATUS    RESTARTS   AGE
pod/beta-app-deployment-8679d5777b-2v77d   1/1     Running   0          95s
pod/prod-app-deployment-586c5dcc59-52wpt   1/1     Running   0          82s
pod/prod-app-deployment-586c5dcc59-sd4h7   1/1     Running   0          82s
pod/prod-app-deployment-586c5dcc59-wc6vq   1/1     Running   0          82s

NAME                                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/beta-app-service-clusterip   ClusterIP   10.100.39.99   <none>        8080/TCP   95s
service/prod-app-service-clusterip   ClusterIP   10.100.33.43   <none>        8080/TCP   82s

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/beta-app-deployment   1/1     1            1           95s
deployment.apps/prod-app-deployment   3/3     3            3           82s

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/beta-app-deployment-8679d5777b   1         1         1       95s
replicaset.apps/prod-app-deployment-586c5dcc59   3         3         3       82s
```

### 設定 Ingress
建立 `ingress-path-aws-eks.yaml`：
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-path
  namespace: app-ns
  annotations: # 加入這個區塊，然後根據各個不同 cloud provider 提供的規格進行客製化
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb # 這裡要改成 alb
  rules:
  - host: all.demo.com
    http:
      paths:
      - pathType: Prefix
        path: /beta
        backend:
          service:
            name: beta-app-service-clusterip
            port:
              number: 8080
      - pathType: Prefix
        path: /prod
        backend:
          service:
            name: prod-app-service-clusterip
            port:
              number: 8080
```

### 部署和測試 Ingress
```bash
$ kubectl apply -f ingress-path-aws-eks.yaml
ingress.networking.k8s.io/ingress-path created

$ kubectl get ingress -n app-ns
NAME           CLASS   HOSTS          ADDRESS                                                                  PORTS   AGE
ingress-path   alb     all.demo.com   k8s-appns-ingressp-943bcaf159-9412560.ap-northeast-1.elb.amazonaws.com   80      30s
# 此處看到 address 表示 ALB 正在部署

# 要確定 ingress load balancer 是否真的建立完畢，要用以下這條：
$ dig +short k8s-appns-ingressp-943bcaf159-9412560.ap-northeast-1.elb.amazonaws.com
35.73.232.178
# 替換成上面看到的 address，此處要等大約 2~3 分鐘才會看到 IP 出現，代表 ALB 部署完成了

$ INGRESS_IP=k8s-appns-ingressp-943bcaf159-9412560.ap-northeast-1.elb.amazonaws.com

$ curl ${INGRESS_IP}:80/beta -H 'Host: all.demo.com'
[beta] served by: beta-app-deployment-8679d5777b-2v77d

$ curl ${INGRESS_IP}:80/prod -H 'Host: all.demo.com'
[prod] served by: prod-app-deployment-586c5dcc59-wc6vq
```
- **跟 local 的 minikube 不同的地方：**
  - Ingress 的 yaml 設定中，`ingressClassName` 要指定為 `alb`
  - 根據規格，在 metadata 之下加入 `annotations`，然後按需求設定

---

## HPA 自動擴展設定

```bash
$ kubectl get pod --all-namespaces
NAMESPACE     NAME                                            READY   STATUS    RESTARTS   AGE
app-ns        beta-app-deployment-8679d5777b-2v77d            1/1     Running   0          20m
app-ns        prod-app-deployment-586c5dcc59-52wpt            1/1     Running   0          20m
app-ns        prod-app-deployment-586c5dcc59-sd4h7            1/1     Running   0          20m
app-ns        prod-app-deployment-586c5dcc59-wc6vq            1/1     Running   0          20m
kube-system   aws-load-balancer-controller-5b895c4dd6-7g2cm   1/1     Running   0          32m
kube-system   aws-load-balancer-controller-5b895c4dd6-lt7r4   1/1     Running   0          32m
kube-system   coredns-5ff77f65d4-4zv6x                        1/1     Running   0          69m
kube-system   coredns-5ff77f65d4-w9tjw                        1/1     Running   0          69m
kube-system   metrics-server-6f674c86fd-859m7                 0/1     Pending   0          71m
kube-system   metrics-server-6f674c86fd-qv6d4                 0/1     Pending   0          71m
```
- 這裡是無法看到 pod 到底是建立在哪個實際的 node 之上的，所以可以加上一個參數，如下：

```bash
$ kubectl get pod -o wide --all-namespaces
NAMESPACE     NAME                                            READY   STATUS    RESTARTS   AGE   IP                NODE                                                         NOMINATED NODE   READINESS GATES
app-ns        beta-app-deployment-8679d5777b-2v77d            1/1     Running   0          22m   192.168.165.38    fargate-ip-192-168-165-38.ap-northeast-1.compute.internal    <none>           <none>
app-ns        prod-app-deployment-586c5dcc59-52wpt            1/1     Running   0          22m   192.168.107.160   fargate-ip-192-168-107-160.ap-northeast-1.compute.internal   <none>           <none>
app-ns        prod-app-deployment-586c5dcc59-sd4h7            1/1     Running   0          22m   192.168.120.84    fargate-ip-192-168-120-84.ap-northeast-1.compute.internal    <none>           <none>
app-ns        prod-app-deployment-586c5dcc59-wc6vq            1/1     Running   0          22m   192.168.138.12    fargate-ip-192-168-138-12.ap-northeast-1.compute.internal    <none>           <none>
kube-system   aws-load-balancer-controller-5b895c4dd6-7g2cm   1/1     Running   0          33m   192.168.150.93    fargate-ip-192-168-150-93.ap-northeast-1.compute.internal    <none>           <none>
kube-system   aws-load-balancer-controller-5b895c4dd6-lt7r4   1/1     Running   0          33m   192.168.143.188   fargate-ip-192-168-143-188.ap-northeast-1.compute.internal   <none>           <none>
kube-system   coredns-5ff77f65d4-4zv6x                        1/1     Running   0          70m   192.168.136.211   fargate-ip-192-168-136-211.ap-northeast-1.compute.internal   <none>           <none>
kube-system   coredns-5ff77f65d4-w9tjw                        1/1     Running   0          70m   192.168.108.57    fargate-ip-192-168-108-57.ap-northeast-1.compute.internal    <none>           <none>
kube-system   metrics-server-6f674c86fd-859m7                 0/1     Pending   0          73m   <none>            <none>                                                       <none>           <none>
kube-system   metrics-server-6f674c86fd-qv6d4                 0/1     Pending   0          73m   <none>            <none>                                                       <none>           <none>
```
- 專門觀察一下前面部署的 pod：
```bash
$ kubectl get pod -o wide -n app-ns
NAME                                   READY   STATUS    RESTARTS   AGE   IP                NODE                                                         NOMINATED NODE   READINESS GATES
beta-app-deployment-8679d5777b-2v77d   1/1     Running   0          24m   192.168.165.38    fargate-ip-192-168-165-38.ap-northeast-1.compute.internal    <none>           <none>
prod-app-deployment-586c5dcc59-52wpt   1/1     Running   0          23m   192.168.107.160   fargate-ip-192-168-107-160.ap-northeast-1.compute.internal   <none>           <none>
prod-app-deployment-586c5dcc59-sd4h7   1/1     Running   0          23m   192.168.120.84    fargate-ip-192-168-120-84.ap-northeast-1.compute.internal    <none>           <none>
prod-app-deployment-586c5dcc59-wc6vq   1/1     Running   0          23m   192.168.138.12    fargate-ip-192-168-138-12.ap-northeast-1.compute.internal    <none>           <none>
```
- 可以看到每個 pod 都被分布在不同的 node 上了，相較之下，在 local 的 minikube 環境裡，大家都只會出現在同一個 node，也就是 local 上
- 這其實也意味著，在真實環境裡，EKS 會協助進行 scaling 相關的動作，可以按照流量門檻讓 cluster 進行伸縮

### 安裝 Metrics Server
```bash
$ cd ~/k8sOnCloud_hiskio/aws_eks/initial/

$ kubectl apply -f metrics-server.yaml 
serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader unchanged
clusterrole.rbac.authorization.k8s.io/system:metrics-server unchanged
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader unchanged
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator unchanged
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server unchanged
service/metrics-server created
deployment.apps/metrics-server created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io unchanged

# 如果有看到如下錯誤訊息：
# The Deployment "metrics-server" is invalid:
# * spec.template.spec.containers[0].ports[1].name: Duplicate value: "https"
# * spec.selector: Invalid value: v1.LabelSelector{MatchLabels:map[string]string{"app.kubernetes.io/instance":"metrics-server", "app.kubernetes.io/name":"metrics-server", "k8s-app":"metrics-server"}, MatchExpressions:[]v1.LabelSelectorRequirement(nil)}: field is immutable
# 
# 表示可能無意間重複安裝了，執行以下指令砍乾淨：
# $ kubectl delete deployment metrics-server -n kube-system
# $ kubectl delete service metrics-server -n kube-system
# $ kubectl delete serviceaccount metrics-server -n kube-system
# 然後再重新安裝一次應該就可以了

$ kubectl get deployment metrics-server -n kube-system -w
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
metrics-server   0/1     1            0           23s
metrics-server   1/1     1            1           72s
```

### 建立 HPA
```bash
$ kubectl autoscale deployment beta-app-deployment --cpu-percent=10 --min=1 --max=10 -n app-ns
horizontalpodautoscaler.autoscaling/beta-app-deployment autoscaled

$ kubectl get hpa -n app-ns
NAME                  REFERENCE                        TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%   1         10        1          3m59s
```
- 剛建立完畢，還很涼

### 監控和測試自動擴展
```bash
# 第一個 tab 用來監控 Pod 和 Node 數量
$ kubectl get pod -o wide -n app-ns -w | grep beta-app

# 第二個 tab 用來查看結果
$ kubectl get hpa -n app-ns -w
# - CPU 使用率百分比
# - 建立的 Pod/Node 數量

# 第三個 tab 用來測試自動擴展
$ curl ${INGRESS_IP}:80/beta -H 'Host: all.demo.com'
$ while sleep 0.005; do curl -s ${INGRESS_IP}:80/beta -H 'Host: all.demo.com'; done
# 按 Ctrl + C 停止

```

- 第一個 tab：

```bash
$ kubectl get pod -o wide -n app-ns -w | grep beta-app
beta-app-deployment-8679d5777b-2v77d   1/1     Running   0          41m   192.168.165.38    fargate-ip-192-168-165-38.ap-northeast-1.compute.internal    <none>           <none>
beta-app-deployment-8679d5777b-gsg4z   0/1     Pending   0          0s    <none>            <none>                                                       <none>           <none>
beta-app-deployment-8679d5777b-gsg4z   0/1     Pending   0          1s    <none>            <none>                                                       8b64cb96b1-e9e17cd094e34d6f90c185b144b3c91c   <none>
beta-app-deployment-8679d5777b-gsg4z   0/1     Pending   0          52s   <none>            fargate-ip-192-168-185-79.ap-northeast-1.compute.internal    8b64cb96b1-e9e17cd094e34d6f90c185b144b3c91c   <none>
beta-app-deployment-8679d5777b-gsg4z   0/1     ContainerCreating   0          53s   <none>            fargate-ip-192-168-185-79.ap-northeast-1.compute.internal    <none>                                        <none>
beta-app-deployment-8679d5777b-gsg4z   1/1     Running             0          79s   192.168.185.79    fargate-ip-192-168-185-79.ap-northeast-1.compute.internal    <none>                                        <none>

# 長出來了，可以先取消然後再重下一次指令比較好觀察：

$ kubectl get pod -o wide -n app-ns -w | grep beta-app
beta-app-deployment-8679d5777b-2v77d   1/1     Running   0          50m     192.168.165.38    fargate-ip-192-168-165-38.ap-northeast-1.compute.internal    <none>           <none>
beta-app-deployment-8679d5777b-gsg4z   1/1     Running   0          2m25s   192.168.185.79    fargate-ip-192-168-185-79.ap-northeast-1.compute.internal    <none>           <none>

# 可以觀察到新長出來的 pod 是生在不同的 node 上
```

- 第二個 tab:

```bash
NAME                  REFERENCE                        TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%   1         10        1          4m37s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%   1         10        1          5m15s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 20%/10%   1         10        1          6m
beta-app-deployment   Deployment/beta-app-deployment   cpu: 22%/10%   1         10        2          6m15s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 21%/10%   1         10        2          6m31s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 19%/10%   1         10        2          6m46s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 19%/10%   1         10        2          7m1sbeta-app-deployment   Deployment/beta-app-deployment   cpu: 17%/10%   1         10        2          7m31s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 10%/10%   1         10        2          7m46s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 11%/10%   1         10        2          8m1s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 10%/10%   1         10        2          8m16s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 11%/10%   1         10        2          9m1s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 6%/10%    1         10        2          9m16s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%    1         10        2          9m31s
beta-app-deployment   Deployment/beta-app-deployment   cpu: 1%/10%    1         10        2          14m
beta-app-deployment   Deployment/beta-app-deployment   cpu: 0%/10%    1         10        1          14m

# 等 CPU usage 到達 20% 左右就可以把第三個 tab 的 curl command 關掉了，然後等個幾分鐘讓 replicas 縮減回 1
```

- 回到第一個 tab：

```bash
$ kubectl get pod -o wide -n app-ns -w | grep beta-app
beta-app-deployment-8679d5777b-2v77d   1/1     Running   0          57m   192.168.165.38    fargate-ip-192-168-165-38.ap-northeast-1.compute.internal    <none>           <none>
```
- 變回一個了

---

## 資源清理

完成測試後，請依序清理以下資源：

1. **刪除 EFS 掛載點**
2. **刪除 EFS 檔案系統**
3. **刪除 EKS 叢集**
    ```bash
    $ eksctl delete cluster --name ${CLUSTER_NAME}
    2025-07-11 15:16:19 [ℹ]  deleting EKS cluster "my-cluster-001"
    2025-07-11 15:16:19 [ℹ]  deleting Fargate profile "app-fp"
    2025-07-11 15:18:28 [ℹ]  deleted Fargate profile "app-fp"
    2025-07-11 15:18:28 [ℹ]  deleting Fargate profile "fp-default"
    2025-07-11 15:20:36 [ℹ]  deleted Fargate profile "fp-default"
    2025-07-11 15:20:36 [ℹ]  deleted 2 Fargate profile(s)
    2025-07-11 15:20:36 [✔]  kubeconfig has been updated
    2025-07-11 15:20:36 [ℹ]  cleaning up AWS load balancers created by Kubernetes objects of Kind Service or Ingress
    Error: deadline surpassed waiting for AWS load balancers to be deleted: k8s-appns-ingressp-943bcaf159

    # 這步驟可以砍掉所有的 fargate profile，然後 ALB 砍不動的話就回 AWS console 手動砍就好
    ```
4. **進入 AWS EC2 -> Load balancers -> 手動刪除 ALB**
5. **進入 AWS VPC -> NAT Gateway -> 手動刪除 Cluster 的 NAT Gateway**
5. **進入 AWS EKS -> Clusters -> 手動刪除 Cluster**
5. **進入 AWS CloudFormation -> Stacks -> 手動刪除 Load balancer controller 以及 EKS Cluster**
5. **刪除管理用 EC2 執行個體**
6. **刪除管理用 VPC**

如果不慎在刪除 EKS 之前先將 VPC 刪除了，可以透過以下順序清除資源：
1. **到 EKS 之下，刪除 Cluster 底下的 Fargate profile**
2. **到 CloudFormation 之下刪除 Load balancer controller 與 Cluster**
3. **回到 VPC 把剩餘的 EKS 相關資源刪除（通常都會在前一步驟清乾淨）**
4. 如果覺得 AWS 卡住砍不動了，就自己按照資源的順序手動全部砍掉

---

## 注意事項

- 所有指令都假設在 Linux 環境下執行
- 請根據實際環境調整變數值（如 VPC ID、EFS 檔案系統 ID 等）
- 某些操作可能需要等待幾分鐘才能完成
