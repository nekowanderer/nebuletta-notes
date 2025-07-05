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

## EKS 叢集建立

### 建立 Fargate 叢集
```bash
$ eksctl create cluster --name ${CLUSTER_NAME} --version 1.28 --fargate
$ cat .kube/config
$ kubectl get all -n kube-system
```

### 建立 OIDC Provider
```bash
$ eksctl utils associate-iam-oidc-provider --cluster ${CLUSTER_NAME} --approve
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


