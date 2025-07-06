# AWS EKS 實作指南

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

以上設定都已經封裝在 [ssm_user_setup.sh](https://github.com/nekowanderer/nebuletta/blob/main/scripts/ssm_user_setup.sh) 腳本之中，可調整 `Configurations` 區塊的變數後直接在 EC2 上執行，相關說明可參考此[連結](https://github.com/nekowanderer/nebuletta/blob/main/scripts/ssm_user_setup.sh)。

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

### 1. 建立 Security Group
- 名稱：`eks-efs-sg`
- 選擇 EKS 叢集的 VPC
- 允許全部連線

### 2. 建立 EFS 檔案系統
- 名稱：`eks-efs`
- 選擇 EKS 叢集的 VPC
- 更新掛載點，使用 Security Group：`eks-efs-sg`
- 等待掛載點可用

### 3. 安裝 EFS CSI Driver
```bash
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/deploy/kubernetes/base/csidriver.yaml
$ kubectl get csidriver
```

### 4. 建立 Persistent Volume (PV)
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
    volumeHandle: fs-0cb4114cfe67fb5c5  # 替換為您的 EFS 檔案系統 ID
```

### 5. 部署 PV 和 PVC
```bash
$ kubectl apply -f aws-efs-volume-pv.yaml
$ kubectl get pv
$ kubectl describe pv app-pv

$ kubectl apply -f simple-volume-pvc.yaml
$ kubectl get pvc
$ kubectl describe pvc app-pvc
```

### 6. 部署應用程式
```bash
$ kubectl apply -f simple-deployment-volume.yaml
$ kubectl get deployments
$ kubectl get pods -w
```

### 7. 測試持久化儲存
```bash
$ kubectl get pods
$ kubectl exec -it [pod_name] -- touch /app/data/file003.txt
$ kubectl exec -it [pod_name] -- ls /app/data

$ kubectl delete pods --all
$ kubectl get pods -w

$ kubectl exec -it [pod_name] -- ls /app/data
```

---

## Load Balancer Controller 設定

### 1. 建立 IAM Policy
```bash
$ curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

$ aws iam create-policy \
   --policy-name AWSLoadBalancerControllerIAMPolicy \
   --policy-document file://iam_policy.json
```

### 2. 建立 Service Account
```bash
$ eksctl create iamserviceaccount \
  --cluster=${CLUSTER_NAME} \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::${AWS_ACCOUNT}:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve

$ eksctl get iamserviceaccount --cluster ${CLUSTER_NAME} --name aws-load-balancer-controller --namespace kube-system
```

### 3. 安裝 Load Balancer Controller
```bash
$ helm repo add eks https://aws.github.io/eks-charts
$ kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"

$ aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.resourcesVpcConfig.vpcId" --output text
$ VPC_ID=vpc-03b06130431a4cb02  # 替換為您的 VPC ID
$ echo ${VPC_ID}

$ helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
    --set clusterName=${CLUSTER_NAME} \
    --set serviceAccount.create=false \
    --set region=${AWS_REGION} \
    --set vpcId=${VPC_ID} \
    --set serviceAccount.name=aws-load-balancer-controller \
    -n kube-system

$ kubectl get all -n kube-system | grep aws-load-balancer
```

---

## 多階段部署專案

### 1. 建立 Fargate Profile
```bash
$ eksctl create fargateprofile --cluster ${CLUSTER_NAME} --region ${AWS_REGION} --name app-fp --namespace app-ns
# 等待幾秒鐘
```

### 2. 部署應用程式
```bash
$ git clone https://github.com/uopsdod/k8sOnCloud_hiskio.git
$ cd k8sOnCloud_hiskio/aws_eks/initial

$ kubectl apply -f beta-app-all-hpa.yaml
$ kubectl apply -f prod-app-all.yaml
$ kubectl get all -n app-ns
```

### 3. 設定 Ingress
建立 `ingress-path-aws-eks.yaml`：
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-path
  namespace: app-ns
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
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

### 4. 部署和測試 Ingress
```bash
$ kubectl apply -f ingress-path-aws-eks.yaml
$ kubectl get ingress -n app-ns
$ kubectl get ingress -n app-ns -w

$ dig +short k8s-appns-ingressp-943bcaf159-450980881.us-west-2.elb.amazonaws.com
# 等待 ADDRESS 出現

$ INGRESS_IP=k8s-appns-ingressp-10f878a9e8-1234487730.us-west-2.elb.amazonaws.com
$ dig +short ${INGRESS_IP}
# 等待約 5 分鐘

$ curl ${INGRESS_IP}:80/beta -H 'Host: all.demo.com'
$ curl ${INGRESS_IP}:80/prod -H 'Host: all.demo.com'
```

---

## HPA 自動擴展設定

### 1. 安裝 Metrics Server
```bash
$ cd ~/k8sOnCloud_hiskio/aws_eks/initial/
$ kubectl apply -f metrics-server.yaml 
$ kubectl get deployment metrics-server -n kube-system 
$ kubectl get deployment metrics-server -n kube-system -w
# 等待幾分鐘
```

### 2. 建立 HPA
```bash
$ kubectl autoscale deployment beta-app-deployment --cpu-percent=10 --min=1 --max=10 -n app-ns
$ kubectl get hpa -n app-ns
$ kubectl get hpa -n app-ns -w
```

### 3. 監控和測試自動擴展
```bash
# 監控 Pod 和 Node 數量
$ kubectl get pod --all-namespaces
$ kubectl get pod -o wide --all-namespaces
$ kubectl get pod -o wide -n app-ns
$ kubectl get pod -o wide -n app-ns -w | grep beta-app

# 測試自動擴展
$ kubectl get ingress -n app-ns
$ INGRESS_IP=k8s-appns-ingressp-10f878a9e8-1234487730.us-west-2.elb.amazonaws.com
$ curl ${INGRESS_IP}:80/beta -H 'Host: all.demo.com'
$ while sleep 0.005; do curl -s ${INGRESS_IP}:80/beta -H 'Host: all.demo.com'; done
# 按 Ctrl + C 停止

# 查看結果
# - CPU 使用率百分比
# - 建立的 Pod/Node 數量
```

---

## 資源清理

完成測試後，請依序清理以下資源：

1. **刪除 EFS 掛載點**
2. **刪除 EFS 檔案系統**
3. **刪除負載平衡器**
4. **刪除 EKS 叢集**
   ```bash
   $ eksctl delete cluster --name ${CLUSTER_NAME}
   ```
5. **刪除管理用 EC2 執行個體**
6. **刪除管理用 VPC**

---

## 注意事項

- 所有指令都假設在 Linux 環境下執行
- 請根據您的實際環境調整變數值（如 VPC ID、EFS 檔案系統 ID 等）
- 某些操作可能需要等待幾分鐘才能完成
- 建議在測試環境中先進行驗證，再部署到生產環境


