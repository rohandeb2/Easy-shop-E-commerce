# 📦 EasyShop — Deployment Guide

<div align="center">

[![AWS](https://img.shields.io/badge/AWS-232F3E?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/)
[![Terraform](https://img.shields.io/badge/Terraform-7B42BC?style=for-the-badge&logo=terraform&logoColor=white)](https://www.terraform.io/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=for-the-badge&logo=jenkins&logoColor=white)](https://www.jenkins.io/)
[![ArgoCD](https://img.shields.io/badge/Argo_CD-EF7B4D?style=for-the-badge&logo=argo&logoColor=white)](https://argoproj.github.io/cd/)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com/)

</div>

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Option A — Local (Docker Compose)](#option-a--local-docker-compose)
4. [Option B — Production (AWS EKS)](#option-b--production-aws-eks)
   - [Step 1 — Provision Infrastructure with Terraform](#step-1--provision-infrastructure-with-terraform)
   - [Step 2 — Configure Jenkins CI](#step-2--configure-jenkins-ci)
   - [Step 3 — Deploy ArgoCD (CD)](#step-3--deploy-argocd-cd)
   - [Step 4 — Ingress & TLS](#step-4--ingress--tls)
5. [Kubernetes Manifests Reference](#kubernetes-manifests-reference)
6. [Environment Variables Reference](#environment-variables-reference)
7. [Troubleshooting](#troubleshooting)

---

## Prerequisites

Install and configure the following tools before proceeding:

| Tool | Purpose | Install |
|---|---|---|
| [![AWS CLI](https://img.shields.io/badge/AWS_CLI-232F3E?style=flat-square&logo=amazon-aws&logoColor=white)](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) | Interact with AWS services | See below |
| [![Terraform](https://img.shields.io/badge/Terraform-7B42BC?style=flat-square&logo=terraform&logoColor=white)](https://developer.hashicorp.com/terraform/install) | Provision cloud infrastructure | See below |
| [![kubectl](https://img.shields.io/badge/kubectl-326CE5?style=flat-square&logo=kubernetes&logoColor=white)](https://kubernetes.io/docs/tasks/tools/) | Manage Kubernetes clusters | `snap install kubectl --classic` |
| [![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)](https://docs.docker.com/get-docker/) | Build & run containers | [docs.docker.com](https://docs.docker.com/get-docker/) |
| [![Helm](https://img.shields.io/badge/Helm-0F1689?style=flat-square&logo=helm&logoColor=white)](https://helm.sh/docs/intro/install/) | Kubernetes package manager | `snap install helm --classic` |

### Install AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

Configure with your IAM credentials:

```bash
aws configure
# AWS Access Key ID: <your-access-key>
# AWS Secret Access Key: <your-secret-key>
# Default region name: eu-west-1
# Default output format: json
```

> **Note:** The IAM user must have permissions for EC2, EKS, VPC, and IAM.

### Install Terraform

```bash
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install terraform -y
terraform -v
```

---

## Option A — Local (Docker Compose)

Best for development and testing. No AWS account required.

### 1. Clone the Repository

```bash
git clone https://github.com/rohandeb2/Easy-shop-E-commerce.git
cd Easy-shop-E-commerce
```

### 2. Create Environment File

```bash
cp .env .env.local
```

Generate secure secrets and update `.env.local`:

```bash
# Generate NEXTAUTH_SECRET
openssl rand -base64 32

# Generate JWT_SECRET
openssl rand -hex 32
```

```env
MONGODB_URI=mongodb://easyshop-mongodb:27017/easyshop
NEXTAUTH_URL=http://localhost:3000
NEXT_PUBLIC_API_URL=http://localhost:3000/api
NEXTAUTH_SECRET=<generated-value>
JWT_SECRET=<generated-value>
```

### 3. Start the Stack

```bash
# Start all services (MongoDB → Migration → App)
docker compose up -d

# Follow logs
docker compose logs -f

# Stop
docker compose down
```

Services start in dependency order:
1. **mongodb** — health-checked before dependents proceed
2. **migration** — seeds the database, exits on completion
3. **app** — starts only after migration succeeds

### 4. Access the Application

Open [http://localhost:3000](http://localhost:3000)

### Useful Commands

```bash
docker ps                          # running containers
docker logs easyshop               # app logs
docker logs easyshop-mongodb       # database logs
docker compose down -v             # stop + remove volumes
```

---

## Option B — Production (AWS EKS)

Full production deployment: Terraform → Jenkins CI → ArgoCD CD → EKS.

---

### Step 1 — Provision Infrastructure with Terraform

The Terraform code provisions:
- **VPC** — 2 public, 2 private, 2 intra subnets across `eu-west-1a/b`
- **EKS Cluster** — `tws-eks-cluster`, SPOT `t2.large` nodes (min 2, max 3)
- **EC2 instance** — `t2.medium` Jenkins server with all tools pre-installed

#### 1.1 Clone & Enter Terraform Directory

```bash
git clone https://github.com/rohandeb2/Easy-shop-E-commerce.git
cd Easy-shop-E-commerce/terraform
```

#### 1.2 Generate SSH Key Pair

```bash
ssh-keygen -f terra-key
chmod 400 terra-key
```

#### 1.3 Apply Infrastructure

```bash
terraform init
terraform plan
terraform apply
# Type 'yes' when prompted
```

Terraform outputs:

```
public_ip               = "<jenkins-ec2-ip>"
eks_cluster_name        = "tws-eks-cluster"
eks_cluster_endpoint    = "https://..."
eks_node_group_public_ips = [...]
```

#### 1.4 SSH into Jenkins Server

```bash
ssh -i terra-key ubuntu@<public_ip>
```

#### 1.5 Update kubeconfig

Run this on the Jenkins server **and** any machine you want to manage EKS from:

```bash
aws configure   # enter credentials
aws eks --region eu-west-1 update-kubeconfig --name tws-eks-cluster
kubectl get nodes   # verify cluster access
```

---

### Step 2 — Configure Jenkins CI

#### 2.1 Access Jenkins

```
http://<public_ip>:8080
```

Get the initial admin password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

If Jenkins is not running:

```bash
sudo systemctl enable jenkins
sudo systemctl restart jenkins
sudo systemctl status jenkins
```

#### 2.2 Install Required Plugins

Navigate to **Manage Jenkins → Plugins → Available Plugins** and install:

- `Docker Pipeline`
- `Pipeline View`

#### 2.3 Configure Credentials

Go to **Manage Jenkins → Credentials → (Global) → Add Credentials**:

| ID | Kind | Usage |
|---|---|---|
| `github-credentials` | Username with password | GitHub repo access |
| `docker-hub-credentials` | Username with password | Push images to DockerHub |

#### 2.4 Configure Shared Library

Go to **Manage Jenkins → Configure System → Global Pipeline Libraries**:

| Field | Value |
|---|---|
| Name | `shared` |
| Default Version | `main` |
| Project Repository URL | `https://github.com/<your-username>/jenkins-shared-libraries` |

> The library must have a `vars/` directory. Fork the shared-lib repo and update `vars/update_k8s_manifest.groovy` with your DockerHub username.

#### 2.5 Create Pipeline Job

1. **New Item** → Name: `EasyShop` → Type: **Pipeline** → OK
2. In **General**: check `GitHub project`, set repo URL
3. In **Trigger**: check `GitHub hook trigger for GITScm polling`
4. In **Pipeline**: set `Pipeline script from SCM` → Git → your fork URL → branch `master` → script path `Jenkinsfile`

#### 2.6 Update Jenkinsfile

Open `Jenkinsfile` in your fork and update the DockerHub username:

```groovy
DOCKER_IMAGE_NAME = 'YOUR_DOCKERHUB_USERNAME/easyshop-app'
DOCKER_MIGRATION_IMAGE_NAME = 'YOUR_DOCKERHUB_USERNAME/easyshop-migration'
```

#### 2.7 Configure GitHub Webhook

In your GitHub repo: **Settings → Webhooks → Add webhook**

```
Payload URL: http://<jenkins-ip>:8080/github-webhook/
Content type: application/json
Trigger: Just the push event
```

#### 2.8 Pipeline Stages

The `Jenkinsfile` runs these stages:

```
Clone Repository
    ↓
Build Docker Images (parallel)
├── Main App Image
└── Migration Image
    ↓
Run Unit Tests
    ↓
Security Scan (Trivy)
    ↓
Push Docker Images (parallel)
├── Push App Image
└── Push Migration Image
    ↓
Update Kubernetes Manifests
```

Click **Build Now** to trigger the first run.

---

### Step 3 — Deploy ArgoCD (CD)

Run the following on the **Bastion server** (EC2 with EKS kubeconfig configured).

#### 3.1 Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Watch pods come up
watch kubectl get pods -n argocd
```

#### 3.2 Expose ArgoCD UI

```bash
# Patch to NodePort
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

# Or port-forward
kubectl port-forward svc/argocd-server -n argocd 8443:443 --address=0.0.0.0 &
```

Access at: `https://<bastion-ip>:8443`

#### 3.3 Get Admin Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

Log in with `admin` / `<decoded-password>`, then change the password via **User Info → Update Password**.

#### 3.4 Create Application in ArgoCD GUI

Click **New App** and fill in:

| Field | Value |
|---|---|
| Application Name | `easyshop` |
| Project | `default` |
| Sync Policy | `Automatic` |
| Repo URL | your fork URL |
| Path | `kubernetes` |
| Cluster URL | `https://kubernetes.default.svc` |
| Namespace | `easyshop` |

Click **Create**. ArgoCD will sync all manifests from the `kubernetes/` directory.

---

### Step 4 — Ingress & TLS

#### 4.1 Install NGINX Ingress Controller

```bash
kubectl create namespace ingress-nginx

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.service.type=LoadBalancer

# Wait for pods
kubectl get pods -n ingress-nginx

# Get the LoadBalancer hostname/IP
kubectl get svc -n ingress-nginx
```

#### 4.2 Install cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.12.0 \
  --set installCRDs=true

kubectl get pods -n cert-manager
```

#### 4.3 Configure DNS

Get your LoadBalancer DNS name:

```bash
kubectl get svc nginx-ingress-ingress-nginx-controller -n ingress-nginx \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

Go to your DNS provider (e.g. GoDaddy) and create a **CNAME record** pointing your domain to this hostname.

#### 4.4 Apply TLS Manifests

Update `kubernetes/04-configmap.yaml` with your domain:

```yaml
NEXT_PUBLIC_API_URL: "https://your-domain.com/api"
NEXTAUTH_URL: "https://your-domain.com/"
```

Update `kubernetes/10-ingress.yaml` with your domain:

```yaml
spec:
  tls:
  - hosts:
    - your-domain.com
    secretName: easyshop-tls
  rules:
  - host: your-domain.com
```

Apply the manifests:

```bash
kubectl apply -f kubernetes/00-cluster-issuer.yaml
kubectl apply -f kubernetes/04-configmap.yaml
kubectl apply -f kubernetes/10-ingress.yaml
```

#### 4.5 Verify TLS Certificate

```bash
# Check certificate issuance
kubectl get certificate -n easyshop
kubectl describe certificate easyshop-tls -n easyshop

# Debug cert-manager
kubectl logs -n cert-manager -l app=cert-manager

# Check ACME challenges
kubectl get challenges -n easyshop
kubectl describe challenges -n easyshop
```

---

## Kubernetes Manifests Reference

All manifests live in `kubernetes/` and are applied in numerical order by ArgoCD.

| Manifest | Kind | Purpose |
|---|---|---|
| `00-cluster-issuer.yaml` | ClusterIssuer | Let's Encrypt ACME issuer |
| `01-namespace.yaml` | Namespace | `easyshop` namespace |
| `02-mongodb-pv.yaml` | PersistentVolume | 5Gi MongoDB storage |
| `03-mongodb-pvc.yaml` | PersistentVolumeClaim | PVC bound to PV |
| `04-configmap.yaml` | ConfigMap | App environment variables |
| `05-secrets.yaml` | Secret | JWT & NextAuth secrets |
| `06-mongodb-service.yaml` | Service | ClusterIP for MongoDB |
| `07-mongodb-statefulset.yaml` | StatefulSet | Single MongoDB replica |
| `08-easyshop-deployment.yaml` | Deployment | 2 replicas of the app |
| `09-easyshop-service.yaml` | Service | NodePort 30000 → pod 3000 |
| `10-ingress.yaml` | Ingress | NGINX ingress with TLS |
| `11-hpa.yaml` | HorizontalPodAutoscaler | Scale 2–5 pods at 70% CPU |
| `12-migration-job.yaml` | Job | One-time DB seed job |

---

## Environment Variables Reference

| Variable | Description | Example |
|---|---|---|
| `MONGODB_URI` | MongoDB connection string | `mongodb://mongodb-service:27017/easyshop` |
| `NEXTAUTH_URL` | Full base URL of the app | `https://your-domain.com` |
| `NEXT_PUBLIC_API_URL` | Public-facing API base URL | `https://your-domain.com/api` |
| `NEXTAUTH_SECRET` | NextAuth signing secret (base64, 32 bytes) | `openssl rand -base64 32` |
| `JWT_SECRET` | JWT signing secret (hex, 32 bytes) | `openssl rand -hex 32` |
| `NODE_ENV` | Runtime environment | `production` |

---

## Troubleshooting

### Docker Compose (Local)

```bash
# MongoDB not reachable
docker ps
docker logs easyshop-mongodb
docker network inspect easyshop-network

# App not accessible
docker logs easyshop
# Ensure port 3000 is free: lsof -i :3000

# Migration failed
docker logs easyshop-migration
# Verify MONGODB_URI in .env.local
```

### Kubernetes / EKS

```bash
# Pod not starting
kubectl get pods -n easyshop
kubectl describe pod <pod-name> -n easyshop
kubectl logs <pod-name> -n easyshop

# MongoDB connection refused
kubectl get svc -n easyshop
kubectl exec -it <app-pod> -n easyshop -- curl mongodb-service:27017

# HPA not scaling
kubectl get hpa -n easyshop
kubectl top pods -n easyshop   # requires metrics-server

# ArgoCD out of sync
kubectl rollout restart deployment easyshop -n easyshop
```

### Jenkins

```bash
# Service not running
sudo systemctl status jenkins
sudo journalctl -u jenkins -n 50

# Clear workspace
# Manage Jenkins → Workspace → Wipe Out
```

### Terraform

```bash
# Provider auth error
aws sts get-caller-identity

# State lock
terraform force-unlock <lock-id>

# Destroy infrastructure
terraform destroy
```

---

![EasyShop Website Screenshot](./public/Deployed.png)

<div align="center">
  <strong>🎉 Your EasyShop is live!</strong>
  <br/><br/>
  <a href="README.md">← Back to README</a>
</div>
