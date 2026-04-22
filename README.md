# EKS Kubernetes Deployment with ALB Ingress Controller

![AWS EKS](https://img.shields.io/badge/AWS-EKS-FF9900?logo=amazonaws&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.27-326CE5?logo=kubernetes&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-ECR-2496ED?logo=docker&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-v3-0F1689?logo=helm&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green)

A production-style Kubernetes deployment on **AWS EKS** — provisioned from scratch with a managed control plane, ECR image registry, ALB Ingress Controller for external traffic routing, and IRSA for secure pod-level AWS access. Deploys a containerised **2048 game app** as a real workload demonstration.

---

## Architecture

```
Internet
    │
    ▼
┌─────────────────────────────────────────┐
│         AWS Application Load Balancer   │
│         (provisioned by ALB Ingress)    │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│              EKS Cluster                │
│                                         │
│  ┌──────────────────────────────────┐   │
│  │         Ingress Resource         │   │
│  │   (routes /  → game-service)     │   │
│  └──────────────┬───────────────────┘   │
│                 │                       │
│  ┌──────────────▼───────────────────┐   │
│  │        game-service (ClusterIP)  │   │
│  └──────────────┬───────────────────┘   │
│                 │                       │
│  ┌──────────────▼───────────────────┐   │
│  │    Deployment (2048 app pods)    │   │
│  │    Image pulled from AWS ECR     │   │
│  └──────────────────────────────────┘   │
│                                         │
│  IAM Roles for Service Accounts (IRSA)  │
└─────────────────────────────────────────┘
```

---

## What's Deployed

| Component | Details |
|---|---|
| EKS Cluster | Managed control plane on AWS (v1.27) |
| Node Group | EC2 worker nodes (t3.medium, auto-scaling) |
| ECR | Private container registry for app image |
| ALB Ingress Controller | Helm-installed; provisions AWS ALB automatically |
| IRSA | IAM Roles for Service Accounts — secure pod AWS access |
| Deployment | 2048 game app — 2 replicas, rolling update strategy |
| Service | ClusterIP — internal routing to pods |
| Ingress | Routes external HTTP traffic via ALB to the app |

---

## Project Structure

```
eks-deployment/
├── manifests/
│   ├── deployment.yaml       # App deployment — image, replicas, resources
│   ├── service.yaml          # ClusterIP service for the app
│   └── ingress.yaml          # ALB Ingress resource with annotations
├── helm/
│   └── alb-controller/       # Helm values for AWS Load Balancer Controller
├── iam/
│   ├── alb-iam-policy.json   # IAM policy for ALB controller
│   └── trust-policy.json     # IRSA trust relationship
├── Dockerfile                # App containerisation
└── README.md
```

---

## Prerequisites

- AWS CLI configured (`aws configure`)
- [eksctl](https://eksctl.io/) installed
- [kubectl](https://kubernetes.io/docs/tasks/tools/) installed
- [Helm v3](https://helm.sh/docs/intro/install/) installed
- IAM user with EKS, EC2, IAM, ECR permissions

---

## Setup & Deployment

### 1. Clone the repo

```bash
git clone https://github.com/Ramana175/eks-deployment.git
cd eks-deployment
```

### 2. Create the EKS Cluster

```bash
eksctl create cluster \
  --name eks-demo \
  --region ap-south-1 \
  --nodegroup-name demo-nodes \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed
```

### 3. Update kubeconfig

```bash
aws eks update-kubeconfig --name eks-demo --region ap-south-1
kubectl get nodes   # verify nodes are Ready
```

### 4. Create ECR repo & push image

```bash
# Create ECR repository
aws ecr create-repository --repository-name 2048-game --region ap-south-1

# Authenticate Docker to ECR
aws ecr get-login-password --region ap-south-1 | \
  docker login --username AWS --password-stdin <account-id>.dkr.ecr.ap-south-1.amazonaws.com

# Build and push
docker build -t 2048-game .
docker tag 2048-game:latest <account-id>.dkr.ecr.ap-south-1.amazonaws.com/2048-game:latest
docker push <account-id>.dkr.ecr.ap-south-1.amazonaws.com/2048-game:latest
```

### 5. Set up IAM for ALB Ingress Controller (IRSA)

```bash
# Create IAM policy
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam/alb-iam-policy.json

# Enable OIDC provider
eksctl utils associate-iam-oidc-provider \
  --region ap-south-1 \
  --cluster eks-demo \
  --approve

# Create IAM service account
eksctl create iamserviceaccount \
  --cluster eks-demo \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::<account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve
```

### 6. Install ALB Ingress Controller via Helm

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=eks-demo \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

### 7. Deploy the application

```bash
kubectl apply -f manifests/deployment.yaml
kubectl apply -f manifests/service.yaml
kubectl apply -f manifests/ingress.yaml
```

### 8. Access the app

```bash
# Get the ALB DNS name
kubectl get ingress -n default

# Open in browser
http://<ALB-DNS-NAME>
```

> Full deployment takes **under 3 minutes** after the cluster is running.

---

## Key Design Decisions

- **ALB Ingress over NodePort/LoadBalancer** — one ALB serves all services via path/host routing, reducing AWS costs
- **IRSA over node-level IAM roles** — pods get only the permissions they need, not everything on the node
- **ECR over DockerHub** — private registry stays within AWS network, faster pulls and no rate limits
- **Rolling update strategy** — zero-downtime deployments with `maxSurge: 1, maxUnavailable: 0`

---

## Cleanup

```bash
kubectl delete -f manifests/
helm uninstall aws-load-balancer-controller -n kube-system
eksctl delete cluster --name eks-demo --region ap-south-1
```

---

## Author

**S. Venkata Ramana**
[LinkedIn](https://linkedin.com/in/venkata-ramana-sanga-176936401) · [GitHub](https://github.com/Ramana175)
