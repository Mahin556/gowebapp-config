# GoWebApp — Cloud-Native Web Application on AWS EKS

> A production-grade Go web application deployed on AWS EKS with full GitOps, secret management, TLS, autoscaling, and CI/CD automation.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Infrastructure](#infrastructure)
- [Secret Management](#secret-management)
- [CI/CD Pipeline](#cicd-pipeline)
- [GitOps with ArgoCD](#gitops-with-argocd)
- [Helm Chart](#helm-chart)
- [Repositories](#repositories)
- [Getting Started](#getting-started)
- [Screenshots](#screenshots)
- [Security-Highlights](#security-highlights)

---

## Overview

**GoWebApp** is a production-ready web application written in Go, designed and deployed following cloud-native best practices on AWS. The project demonstrates a complete end-to-end DevOps workflow — from infrastructure provisioning with Terraform, to container image building, security scanning, Helm chart publishing, and fully automated GitOps-based deployments on Amazon EKS.

The application uses an **AWS RDS MySQL** database as its backend, with all sensitive credentials managed securely through **AWS Secrets Manager** and **SSM Parameter Store**, surfaced into Kubernetes via the **External Secrets Operator (ESO)**. TLS certificates are automatically provisioned and renewed via **cert-manager** with Let's Encrypt.

---

## Architecture

```
                        ┌─────────────────────────────────────────┐
                        │              AWS Cloud (ap-south-1)      │
                        │                                          │
   User ──► Route53 ──► │  ALB ──► NGINX Ingress ──► EKS Cluster  │
                        │                │                         │
                        │         ┌──────┴──────┐                  │
                        │         │  GoWebApp   │                  │
                        │         │    Pods     │                  │
                        │         └──────┬──────┘                  │
                        │                │                         │
                        │         ┌──────▼──────┐                  │
                        │         │  RDS MySQL  │                  │
                        │         │  (Private   │                  │
                        │         │   Subnet)   │                  │
                        │         └─────────────┘                  │
                        │                                          │
                        │  Secrets Manager ◄── ESO ──► K8s Secrets │
                        │  SSM Parameter Store                     │
                        └─────────────────────────────────────────┘

GitHub ──► CI/CD Pipeline ──► DockerHub ──► Helm Chart ──► ArgoCD ──► EKS
```

---

## Tech Stack

| Category | Tool |
|---|---|
| **Language** | Go (Golang) |
| **Container** | Docker (multi-arch: amd64/arm64) |
| **Registry** | DockerHub |
| **Infrastructure** | Terraform |
| **Kubernetes** | Amazon EKS |
| **Database** | AWS RDS MySQL 8.0 |
| **Package Manager** | Helm |
| **GitOps** | ArgoCD |
| **Secret Management** | AWS Secrets Manager + SSM + External Secrets Operator |
| **TLS** | cert-manager + Let's Encrypt |
| **Ingress** | NGINX Ingress Controller |
| **Autoscaling** | Kubernetes HPA |
| **CI/CD** | GitHub Actions |
| **Image Scanning** | Docker Scout |
| **Code Quality** | golangci-lint |
| **Notifications** | Slack |
| **IaC State** | AWS S3 (encrypted, versioned) |

---

## Infrastructure

All infrastructure is provisioned and managed with **Terraform**, with remote state stored in an encrypted, versioned S3 bucket.

### AWS Resources

- **VPC** with public and private subnets across multiple Availability Zones
- **Internet Gateway + NAT Gateway** for outbound traffic from private subnets
- **EKS Cluster** with managed node groups in private subnets
- **RDS MySQL** instance in private subnets with auto-managed master credentials
- **IAM Roles** with least-privilege policies (IRSA for pod-level AWS access)
- **OIDC Provider** for EKS service account federation with AWS IAM
- **Security Groups** for EKS control plane, node group, and RDS

### Networking

```
VPC (10.0.0.0/16)
├── Public Subnets   → ALB, NAT Gateway
└── Private Subnets  → EKS Nodes, RDS (multi-AZ)
```

---

## Secret Management

Secrets are managed using a **zero-credentials** approach — no AWS credentials are stored anywhere in the cluster.

```
AWS Secrets Manager ──┐
                       ├──► External Secrets Operator ──► K8s Secrets ──► App Pods
AWS SSM Parameter  ──┘
Store
```

| Secret | Store | Path |
|---|---|---|
| App DB credentials | Secrets Manager | `gowebapp/prod/database/app_cred` |
| RDS root credentials | Secrets Manager | `rds!db-<uuid>` (auto-managed by RDS) |
| DB host | SSM Parameter Store | `/gowebapp/prod/database/host` |
| DB port | SSM Parameter Store | `/gowebapp/prod/database/port` |

ESO accesses AWS using **IRSA (IAM Roles for Service Accounts)** — the ESO service account is annotated with an IAM role ARN, and AWS grants temporary credentials via OIDC token exchange. No static credentials anywhere.

---

## CI/CD Pipeline

The GitHub Actions pipeline runs automatically on every push to the `modules` branch.

```
Push to GitHub
      │
      ▼
┌─────────────┐
│    Build    │  → go build + go test
└──────┬──────┘
       │
       ▼
┌─────────────────┐
│  Code Quality   │  → golangci-lint
└────────┬────────┘
         │
         ▼
┌──────────────────────┐
│  Docker Build & Push │  → multi-arch (amd64/arm64) → DockerHub
└──────────┬───────────┘
           │
           ▼
┌──────────────────┐
│   Image Scan     │  → Docker Scout (critical/high CVEs)
└────────┬─────────┘
         │
         ▼
┌──────────────────────┐
│   Helm Chart Update  │  → bump version, update image tag, push to gh-pages
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  Slack Notification  │  → pipeline status report
└──────────────────────┘
```

### Key pipeline features

- **Multi-architecture builds** — images support both `linux/amd64` and `linux/arm64`
- **Build cache** — uses Docker layer caching via registry to speed up builds
- **Automatic image tagging** — uses GitHub `run_id` as the unique image tag
- **Security scanning** — Docker Scout blocks deployment on critical/high CVEs
- **Helm auto-versioning** — chart version is auto-incremented on every release
- **Slack alerts** — full pipeline status report with per-job results

---

## GitOps with ArgoCD

ArgoCD continuously monitors the Helm chart repository and automatically syncs any changes to the EKS cluster.

```
Helm chart pushed to gh-pages
           │
           ▼
    ArgoCD detects new version (polls every 3 min)
           │
           ▼
    ArgoCD syncs → rolling update → new pods deployed ✅
```

### ArgoCD Application features

- **Auto-sync** with self-healing — ArgoCD corrects any manual cluster changes
- **Pruning** — removes resources deleted from the Helm chart
- **Retry with backoff** — automatically retries failed syncs (up to 5 times)
- **Sync waves** — ExternalSecrets sync before the application (`wave: -1`)
- **Namespace auto-creation** — ArgoCD creates the `gowebapp` namespace if missing

---

## Helm Chart

The Helm chart is hosted on **GitHub Pages** as a Helm repository and versioned automatically by the CI/CD pipeline.

```bash
helm repo add gowebapp https://mahinraza.github.io/helm-charts
helm repo update
helm install gowebapp gowebapp/gowebapp
```

### Chart components

| Resource | Description |
|---|---|
| `Deployment` | Main GoWebApp application pods |
| `Service` | ClusterIP service on port 80 → 8080 |
| `Ingress` | NGINX ingress with TLS termination |
| `HPA` | Autoscales 1–5 pods on CPU/memory |
| `ConfigMap` | Application configuration (ports, sessions, views) |
| `ExternalSecret` | Pulls secrets from AWS into K8s |
| `ClusterSecretStore` | Connects ESO to AWS Secrets Manager and SSM |
| `Issuer` | cert-manager Let's Encrypt ACME issuer |
| `Job` | One-time DB init/migration job |

---

## Repositories

This project is split across 3 repositories, each with a distinct responsibility:

### 1. [gowebapp](https://github.com/mahin556/gowebapp.git) — Application
The main application repository containing the Go source code and the GitHub Actions CI/CD pipeline. Every push to the `modules` branch triggers the full pipeline — build, test, lint, Docker image build & push, security scan, and Helm chart update.

```bash
.
|-- Dockerfile
|-- main.go
|-- go.mod
|-- go.sum
|-- app/
|   |-- controller/               # Route handlers (index, login, register, notepad, etc.)
|   |-- model/                    # Database models (user, note)
|   |-- route/
|   |   |-- middleware/           # ACL, logging, pprof handlers
|   |   `-- route.go              # Route definitions
|   `-- shared/
|       |-- database/             # DB connection
|       |-- session/              # Session management
|       |-- passhash/             # Password hashing
|       |-- view/                 # Template rendering + plugins
|       `-- server/               # HTTP server setup
|-- config/
|   |-- config.json               # App configuration
|   `-- mysql.sql                 # Database schema
|-- static/                       # CSS, JS, fonts, favicons
|-- template/                     # HTML templates (.tmpl)
|   |-- base.tmpl
|   |-- index/
|   |-- login/
|   |-- register/
|   |-- notepad/
|   `-- partial/                  # Shared header/footer
`-- .github/
    `-- workflows/
        `-- cicd.yml              # GitHub Actions CI/CD pipeline
```

### 2. [gowebapp-config](https://github.com/mahin556/gowebapp-config.git) — Infrastructure & Cluster Config
The configuration repository containing:
- **Terraform** code for all AWS infrastructure (VPC, EKS, RDS, IAM, ECR, Secrets)
- **ArgoCD** Application and AppProject manifests
- **Kubernetes manifests** for cluster-level resources (ESO, cert-manager, ingress controller, etc.)

> Note: Kubernetes manifests are present but not used for application deployment — the application is deployed exclusively via Helm through ArgoCD.

```bash
.
|-- infra/                        # Terraform — all AWS infrastructure
|   |-- network.tf                # VPC, subnets, IGW, NAT Gateway
|   |-- eks.tf                    # EKS cluster and node group
|   |-- db.tf                     # RDS MySQL instance
|   |-- iam.tf                    # IAM roles and policies (IRSA)
|   |-- oidc.tf                   # OIDC provider for EKS
|   |-- eso.tf                    # ESO IAM role and policy
|   |-- secrets.tf                # AWS Secrets Manager resources
|   |-- ssm.tf                    # SSM Parameter Store
|   |-- locals.tf                 # Local values and naming conventions
|   |-- variables.tf              # Input variables
|   |-- output.tf                 # Output values
|   |-- terraform.tf              # Provider and backend config
|   |-- terraform.tfvars          # Variable values
|   `-- secrets.tfvars            # Sensitive variable values (gitignored)
|-- manifests/
|   |-- app-aws/                  # Raw K8s manifests for AWS deployment
|   |   |-- 01-ns.yaml
|   |   |-- 02-app-cred-external-secret.yaml
|   |   |-- 02-configmap.yaml
|   |   |-- 02-db-ep-external-secret.yaml
|   |   |-- 02-root-cred-external-secret.yaml
|   |   |-- 03-external-name.yaml
|   |   |-- 04-init-db-job.yaml
|   |   |-- 05-deployment.yaml
|   |   |-- 06-svc.yaml
|   |   `-- 07-hpa.yaml
|   |-- app-do/                   # Raw K8s manifests for DigitalOcean deployment
|   `-- argocd/                   # ArgoCD Application and AppProject manifests
|       |-- gowebapp-app-aws.yaml
|       |-- gowebapp-app-do.yaml
|       |-- gowebapp-app-helm-aws.yaml     # ✅ Active — Helm-based ArgoCD app
|       |-- gowebapp-app-helm-do.yaml
|       `-- gowebapp-appproject.yaml
`-- clustersecretstore.yaml       # ClusterSecretStore for ESO
```

### 3. [helm-charts](https://github.com/mahin556/helm-charts.git) — Helm Chart Repository
A centralized Helm chart repository hosted on GitHub Pages, designed to store Helm charts for multiple applications in one place. The `gowebapp` Helm chart lives here and is automatically versioned and published by the CI/CD pipeline on every release.

> The `gowebapp-config` repo could also serve as the Helm chart host — using a dedicated repo makes it easier to manage charts across multiple applications from a single source.

```bash
.
|-- index.yaml                    # Helm repo index (auto-generated by CI)
|-- gowebapp/                     # GoWebApp Helm chart
|   |-- Chart.yaml
|   |-- values.yaml
|   `-- templates/
|       `-- app/
|           |-- deployment.yaml
|           |-- service.yaml
|           |-- ingress.yaml
|           |-- hpa.yaml
|           |-- cm.yaml                        # ConfigMap
|           |-- init-db-job.yaml               # DB init job
|           |-- external-service.yaml          # ExternalName service for RDS
|           |-- app-cred-external-secret.yaml  # ESO — app credentials
|           |-- root-cred-external-secret.yaml # ESO — RDS root credentials
|           |-- db-ep-external-secret.yaml     # ESO — DB endpoint
|           |-- issuer.yaml                    # cert-manager Issuer
|           `-- certificate.yaml               # TLS certificate
|-- gowebapp-1.0.0.tgz            # Released chart versions (auto-packaged by CI)
|-- gowebapp-3.0.5.tgz            # Latest release
|-- cymbal_ecommerce/             # Other hosted application charts
|-- go-portfolio-app/
`-- tetris-game/
```

---



## Getting Started

### Prerequisites

- AWS CLI configured with appropriate permissions
- Terraform >= 1.4
- kubectl
- Helm >= 3.x
- ArgoCD CLI

### 1. Create Remote Backend S3 Bucket

```bash
export BUCKET=<S3_BUCKET_NAME>
export REGION=<AWS_REGION_CODE>

aws s3api create-bucket \
  --bucket $BUCKET \
  --region $REGION \
  --create-bucket-configuration LocationConstraint=$REGION

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket $BUCKET \
  --versioning-configuration Status=Enabled

# Block public access
aws s3api put-public-access-block \
  --bucket $BUCKET \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# Enable server-side encryption
aws s3api put-bucket-encryption \
  --bucket $BUCKET \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'
```

### 2. Provision Infrastructure

```bash
# Initialize Terraform
terraform init -var-file=secrets.tfvars

# Review the plan
terraform plan -var-file=secrets.tfvars

# Apply infrastructure
terraform apply -var-file=secrets.tfvars
```

```bash
cat << EOF > terraform.tfvars
region      = "<AWS_REGION>"
profile     = "<AWS_PROFILE>"
environment = "<ENVIRONMENT>"
project     = "<PROJECT_NAME>"

cluster_version     = "1.34"
node_instance_types = ["t3.medium"]
node_disk_size      = 20
node_desired_size   = "2"
node_min_size       = "1"
node_max_size       = "3"

addon_versions = {
  coredns        = "v1.12.3-eksbuild.1" # Latest for K8s 1.34
  kube_proxy     = "v1.34.3-eksbuild.2" # Must match the K8s major.minor version (1.34.x)
  vpc_cni        = "v1.20.4-eksbuild.2" # Latest stable for 1.34
  ebs_csi_driver = "v1.56.0-eksbuild.1" # Compatible with 1.34
}

vpc_cidr = "<VPC_CIDR>"

key_name = "<SSH_KEY_NAME>"

public_subnets = [
  {
    suffix            = "<PUBLIC_SUBNET_NAME>"
    cidr_block        = "<PUBLIC_SUBNET_CIDR>"
    availability_zone = "<AZ>"
  }
]

private_subnets = [
  {
    suffix            = "<FIRST_PRIVATE_SUBNET_NAME>"
    cidr_block        = "<FIRST_PRIVATE_SUBNET_CIDR>"
    availability_zone = "<SECOND_PRIVATE_SUBNET_AZ>"
  },
  {
    suffix            = "<SECOND_PRIVATE_SUBNET_NAME>"
    cidr_block        = "<SECOND_PRIVATE_SUBNET_CIDR>"
    availability_zone = "<FIRSTS_PRIVATE_SUBNET_AZ>"
  }
]

rds = {
  identifier_suffix   = "<RDS_INSTANCE_NAME>"
  engine              = "mysql"
  engine_version      = "8.0"
  instance_class      = "db.t3.micro"
  allocated_storage   = 20
  db_name             = "testdb"
  master_username     = "admin"
  publicly_accessible = false
  skip_final_snapshot = true
}

db_kubernetes_service_name = "mysql-external-service.gowebapp.svc.cluster.local"

EOF
```
```bash
cat << EOF > secrets.tfvars
# db_root_username = "value"
# db_root_password = "value"
db_app_username    = "<APP_DB_USER_NAME>"
db_app_password    = "<APP_DB_USER_PASSWORD"
db_name            = "<APP_DATABASE>"
session_secret_key = "<APP_SECRET_SESSION_KEY>"
account_id         = "<YOUR_AWS_ACCOUNT_ID>"

EOF
```

### 3. Configure kubectl

```bash
export REGION=<AWS_REGION_CODE>
export CLUSTER_NAME=<YOUR_EKS_CLUSTER_NAME>

aws eks update-kubeconfig \
  --name $CLUSTER_NAME \
  --region $REGION
```

### 4. Install External Secrets Operator (ESO)

```bash
helm repo add external-secrets https://charts.external-secrets.io

helm repo ls
helm repo update

# helm install external-secrets \
#    external-secrets/external-secrets \
#     -n external-secrets \
#     --create-namespace \
#     --version 2.1.0 \
#     --set installCRDs=true

# Install ESO with IRSA role
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=$(terraform output -raw eso_role_arn) \
    --set installCRDs=true

# Verify installation
helm list -n external-secrets
kubectl get all -n external-secrets
kubectl get crd | grep external-secrets
```

### 5. Install Metrics Server (for HPA)
```bash
kubectl apply -f metrics-server-v0.8.1.yaml
```

### 6. Install cert-manager
```bash
helm repo add jetstack https://charts.jetstack.io --force-update

helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.20.0 \
  --set crds.enabled=true
```


### 7. Deploy Cluster Secret Store

```bash
export REGION=<AWS_REGION_CODE>
export EXTERNAL_SECRET_NAME=<EXTERNAL_SECRET_NAME>
export EXTERNAL_SECRET_NAMESPACE=<EXTERNAL_SECRET_NAMESPACE>

kubectl apply -f -<<EOF
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: aws-secretsmanager-store
spec:
  provider:
    aws:
      service: SecretsManager
      region: $REGION
      auth:
        jwt:
          serviceAccountRef:
            name: $EXTERNAL_SECRET_NAME
            namespace: $EXTERNAL_SECRET_NAMESPACE

--- 

apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: aws-ssm-store
spec:
  provider:
    aws:
      service: ParameterStore
      region: $REGION
      auth:
        jwt:
          serviceAccountRef:
            name: $EXTERNAL_SECRET_NAME
            namespace: $EXTERNAL_SECRET_NAMESPACE
EOF

```

### 8. Deploy NGINX Ingress Controller

```bash
helm upgrade --install ingress-nginx ingress-nginx   --repo https://kubernetes.github.io/ingress-nginx   --namespace ingress-nginx --create-namespace

kubectl get all -n ingress-nginx
```

### 9. Install ArgoCD CLI

```bash
VERSION=$(curl -L -s https://raw.githubusercontent.com/argoproj/argo-cd/stable/VERSION)
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/download/v$VERSION/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

### 10. Install ArgoCD

```bash
# Add the official Argo CD Helm repository
helm repo add argo https://argoproj.github.io/argo-helm

# Verify that the repository was added successfully
helm repo list

# Similar to `apt update`, refreshes local cache of chart versions from all added Helm repositories
helm repo update

# Search for Argo-related charts across all added repositories
helm search repo argo

# List available versions of the official Argo CD Helm chart
helm search repo argo/argo-cd --versions

kubectl create ns argocd

helm install my-argo-cd argo/argo-cd \
  --version 9.1.9 \
  -n argocd 

helm ls -n argocd

kubectl get all -n argocd

kubectl get svc -n argocd
```

### 11. Access ArgoCD UI
```bash
# Port forward ArgoCD UI
kubectl port-forward service/my-argo-cd-argocd-server -n argocd 8080:443
```


### 12. Get Admin Password
```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

### 13. Login to ArgoCD
```bash
argocd login localhost:8080
```

### 14. Add Git Repository to ArgoCD
```bash
argocd repo add https://github.com/mahinraza/gowebapp-config.git \
  --name gowebapp-config-repo \
  --username mahinraza \
  --password <your-github-pat-token>
```

### 15. Deploy ArgoCD Project

```bash
cd manifests
kubectl apply -f argocd/gowebapp-appproject.yaml
```

### 16. Deploy via ArgoCD

```bash
cd manifests
kubectl apply -f argocd/gowebapp-app-helm-aws.yaml
argocd app sync gowebapp
```

### 17. Verify deployment

```bash
# Check pods
kubectl get pods -n gowebapp

# Check secrets synced
kubectl get externalsecret -n gowebapp

# Check ingress
kubectl get ingress -n gowebapp
```

### Cleanup
```bash
### Destroy infra
terraform destroy

### Delete remote s3 backend
export BUCKET=<S3_BUCKET_NAME>
export REGION=<AWS_REGION_CODE>

# Remove all object versions (required since versioning is enabled)
aws s3api delete-objects \
  --bucket $BUCKET \
  --delete "$(aws s3api list-object-versions \
    --bucket $BUCKET \
    --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}' \
    --output json)"

# Remove all delete markers
aws s3api delete-objects \
  --bucket $BUCKET \
  --delete "$(aws s3api list-object-versions \
    --bucket $BUCKET \
    --query '{Objects: DeleteMarkers[].{Key:Key,VersionId:VersionId}}' \
    --output json)"

# Delete the bucket
aws s3api delete-bucket \
  --bucket $BUCKET \
  --region $REGION
```


---

## Screenshots

![](/images/image.png)
![](/images/image%20(11).png)
![](/images/image%20(1).png)
![](/images/image%20(10).png)
![](/images/image%20(9).png)
![](/images/image%20(7).png)
![](/images/image%20(8).png)
![](/images/image%20(6).png)
![](/images/image%20(4).png)
![](/images/image%20(3).png)
![](/images/image%20(2).png)


---

## Security Highlights

- **Zero static credentials** in cluster — all AWS access via IRSA
- **Secrets never in Git** — pulled from AWS at runtime via ESO
- **Encrypted state** — Terraform state encrypted at rest in S3
- **Private subnets** — EKS nodes and RDS never directly exposed to internet
- **Image scanning** — Docker Scout blocks on critical/high CVEs before deploy
- **Non-root containers** — app runs as UID 1001
- **TLS everywhere** — Let's Encrypt certificates auto-renewed via cert-manager

---

*Maintained by [@mahinraza](https://github.com/mahinraza) · Managed by Terraform · Deployed via ArgoCD*
