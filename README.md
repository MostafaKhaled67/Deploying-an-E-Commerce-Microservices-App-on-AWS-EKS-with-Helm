# Deploying a Microservices E-Commerce App on AWS EKS with Helm

Hands-on project: deploy **RobotShop**, a microservices e-commerce application, on **Amazon EKS** and expose it to the internet through an **Application Load Balancer** created from a Kubernetes Ingress.



---

## Table of Contents

- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Prerequisites](#prerequisites)
- [Deployment Steps](#deployment-steps)

   <img width="736" height="291" alt="image" src="https://github.com/user-attachments/assets/d69034a8-c1f6-4878-96e6-dff59646ab34" />





The app runs as **12 pods** in three tiers: frontend, backend services, and data services.

## Tech Stack

| Area | Tools |
|------|-------|
| Cloud | AWS (EKS, EC2, VPC, ALB, IAM, EBS, CloudFormation) |
| Orchestration | Kubernetes 1.34 |
| Packaging | Helm |
| Provisioning | eksctl |
| Environment | AWS CloudShell |

**AWS components used**

- EKS cluster with a managed node group (`t3.small`)
- VPC with public and private subnets across 2 AZs (created by eksctl)
- IAM OIDC provider for IAM Roles for Service Accounts (IRSA)
- AWS Load Balancer Controller (installed with Helm)
- Amazon EBS CSI Driver (EKS add-on) with its own IAM role

## Prerequisites

- An AWS account and a user with enough permissions (for learning: `AdministratorAccess`)
- Access to **AWS CloudShell** (comes with `aws` and `kubectl`)
- `eksctl` and `helm` (installed in CloudShell, see below)

Install the missing tools in CloudShell:

```bash

mkdir -p ~/bin
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" | tar xz -C ~/bin
export PATH=$PATH:~/bin


curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

## Deployment Steps

Cluster name used here: `ecommerce-cluster`, region: `us-east-1`. Replace them with your own values.

### Step 1: Create the EKS cluster

```bash
eksctl create cluster \
  --name ecommerce-cluster \
  --region us-east-1 \
  --nodegroup-name ecommerce-nodes \
  --node-type t3.small \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3
```

Takes around 15 to 20 minutes. When it finishes:

```bash
kubectl get nodes
```

### Step 2: Configure the IAM OIDC provider

```bash
export cluster_name=ecommerce-cluster
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)

# Check if a provider already exists (empty output = none)
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4

# If not, create it
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
```

### Step 3: Set up the IAM policy and service account for the ALB controller

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

eksctl create iamserviceaccount \
  --cluster=ecommerce-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

> **Heads up:** this policy file is v2.5.4. Newer versions of the controller need extra permissions. See [Problem 2](#2-the-alb-returned-404-on-every-request). Using the latest policy file from the start avoids it.

### Step 4: Install the AWS Load Balancer Controller

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

export VPC_ID=$(aws eks describe-cluster --name ecommerce-cluster --query "cluster.resourcesVpcConfig.vpcId" --output text)

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=ecommerce-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=$VPC_ID

kubectl get deployment -n kube-system aws-load-balancer-controller
```

### Step 5: Configure the EBS CSI Driver

```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster ecommerce-cluster \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --role-only \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve

eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster ecommerce-cluster \
  --service-account-role-arn arn:aws:iam::${ACCOUNT_ID}:role/AmazonEKS_EBS_CSI_DriverRole \
  --force

eksctl get addon --name aws-ebs-csi-driver --cluster ecommerce-cluster
```

### Step 6: Install the RobotShop Helm chart

```bash
git clone https://github.com/uniquesreedhar/RobotShop-Project.git
cd RobotShop-Project/EKS/helm

kubectl create ns robot-shop
helm install robot-shop --namespace robot-shop .

kubectl get pods -n robot-shop
```

### Step 7: Create the Ingress

```bash
cd ~/RobotShop-Project/EKS/helm
kubectl apply -f ingress.yaml
kubectl get ingress -n robot-shop
```

When the `ADDRESS` column is filled, open it in the browser over **http** (not https). This creates an Application Load Balancer in your account.

## Problems I Faced and How I Solved Them

### 1. Pods stuck in `Pending`

**Symptom:** four pods (`user`, `mysql`, `rabbitmq`, `dispatch`) stayed `Pending`, while `shipping` and `ratings` kept restarting.

**Diagnosis:** the scheduler events showed two causes together:

```
0/2 nodes are available: 2 Insufficient memory, 2 Too many pods.
```

A `t3.small` has 2 GB of RAM and a low pod limit per node, and the system pods (CoreDNS, kube-proxy, VPC CNI, EBS CSI, ALB controller, metrics-server) already use a big part of it.

**Fix:** scaled the node group to 3 nodes.

```bash
eksctl scale nodegroup --cluster ecommerce-cluster --name ecommerce-nodes --nodes 3 --region us-east-1
```

The pending pods were scheduled right away. The restarts on `shipping` and `ratings` stopped once MySQL was running, because they depend on it.

### 2. The ALB returned 404 on every request

**Symptom:** the ALB was created and responding, but every request returned `404 Not Found` with `Server: awselb/2.0`, meaning the answer came from the ALB itself, not from the app.

```bash
curl -I http://<alb-dns-name>
```

**Diagnosis:** `ingress.yaml` was correct (no host restriction, path `/`, backend `web:8080`), so the problem was elsewhere. The controller logs showed:

```
AccessDenied ... elasticloadbalancing:DescribeListenerAttributes
```

The IAM policy came from v2.5.4 of the controller, while the Helm chart installed a newer controller that needs more permissions. The controller created the ALB but failed to add the routing rules, so the ALB kept its default 404 response.

**Fix:** created a new version of the policy from the latest policy file, set it as default, then restarted the controller.

```bash
curl -o iam_policy_latest.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

aws iam create-policy-version \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy_latest.json \
  --set-as-default

kubectl rollout restart deployment aws-load-balancer-controller -n kube-system
kubectl rollout status deployment aws-load-balancer-controller -n kube-system
```


