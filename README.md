markdown# Tech Challenge 2 — Application Deployment

A simple "Hello, World!" web app deployed via Docker, Terraform, AWS EKS, and CI/CD — implemented two ways: **Jenkins** (this branch) and **GitOps with GitHub Actions + Argo CD** (`gitops` branch).

## Live Application URL
http://k8s-default-techchal-5ecee2d527-572673933.us-east-1.elb.amazonaws.com

## Project Overview
This project demonstrates:
- A containerized Flask web application
- Infrastructure as Code (Terraform) provisioning a full AWS EKS cluster
- Auto-scaling via Kubernetes HPA (1-3 pods, triggered at 50% CPU/memory)
- Auto-scaling node group (1-4 t3.small nodes)
- Internet-facing access via AWS ALB Ingress
- CI/CD pipeline via Jenkins (main branch)
- CI/CD pipeline via GitHub Actions + Argo CD, GitOps-style (gitops branch)

## Tech Stack
- **App:** Python / Flask
- **Containerization:** Docker
- **Infrastructure:** Terraform, AWS EKS, VPC, ALB, ECR
- **CI/CD (main):** Jenkins
- **CI/CD (gitops):** GitHub Actions + Argo CD + Helm

## Prerequisites
- AWS account with configured credentials (`aws configure`)
- Docker
- Terraform
- kubectl
- Helm
- eksctl
- GitHub CLI (optional, used for auth)

## Setup Instructions

### 1. Clone the repo
gh repo clone LifeasJJ/techchallenge2
cd techchallenge2

### 2. Run the app locally (optional sanity check)
pip install flask --break-system-packages
python3 app.py
curl http://localhost:5000

### 3. Build and test the Docker image
docker build -t techchallenge2 .
docker run -d -p 5000:5000 --name techchallenge2-test techchallenge2
curl http://localhost:5000
docker stop techchallenge2-test && docker rm techchallenge2-test

### 4. Provision AWS infrastructure with Terraform
cd terraform
terraform init
terraform plan
terraform apply
This creates:
- A VPC with public/private subnets across 2 availability zones
- An EKS cluster (`techchallenge2-cluster`), Kubernetes v1.33
- A managed node group: t3.small instances, auto-scaling 1 (min) to 4 (max)

### 5. Connect kubectl to the cluster
aws eks update-kubeconfig --name techchallenge2-cluster --region us-east-1
kubectl get nodes

### 6. Push the app image to ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <your-account-id>.dkr.ecr.us-east-1.amazonaws.com
docker tag techchallenge2:latest <your-account-id>.dkr.ecr.us-east-1.amazonaws.com/techchallenge2:latest
docker push <your-account-id>.dkr.ecr.us-east-1.amazonaws.com/techchallenge2:latest

### 7. Deploy to EKS
cd ../k8s
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f hpa.yaml
kubectl apply -f ingress.yaml
Install the metrics server (required for HPA) and the AWS Load Balancer Controller (required for the ALB Ingress) — see Deployment Steps below for full details.

## Deployment Steps
1. **Metrics Server** — required for HPA to read CPU/memory usage:
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
2. **AWS Load Balancer Controller** — required for the ALB Ingress to provision a real load balancer:
   - Create IAM policy from the official `iam_policy.json`
   - Create an IAM service account via `eksctl create iamserviceaccount`
   - Install via Helm: `helm install aws-load-balancer-controller eks/aws-load-balancer-controller`
3. Apply the Deployment, Service, HPA, and Ingress manifests in `/k8s`
4. Wait 1-3 minutes for the ALB to provision, then retrieve the URL:
kubectl get ingress techchallenge2-ingress

## Terraform Code Explanation
- **provider.tf** — configures the AWS provider and region (us-east-1)
- **vpc.tf** — provisions a VPC using the official `terraform-aws-modules/vpc` module, with public subnets (for the ALB) and private subnets (for worker nodes)
- **eks.tf** — provisions the EKS cluster using the official `terraform-aws-modules/eks` module, with a managed node group scaling 1-4 t3.small nodes, cluster-creator admin access enabled, and KMS/CloudWatch log group creation disabled (to work within available IAM permissions)

## Jenkins Pipeline Explanation
The `Jenkinsfile` defines a 4-stage pipeline that runs on the `main` branch:
1. **Checkout** — pulls the latest code from GitHub
2. **Build Docker Image** — builds the image, tags it with both the Jenkins build number and `latest`
3. **Push to ECR** — authenticates to Amazon ECR using AWS credentials stored in Jenkins' credential store, pushes both tags
4. **Deploy to EKS** — updates kubeconfig, then uses `kubectl set image` to roll out the new image to the live deployment, waiting for the rollout to complete

## GitOps Alternative Explanation (gitops branch)
On the `gitops` branch, deployment follows a GitOps pattern instead of Jenkins:
- **`techchallenge2-chart/`** — a Helm chart packaging the Deployment, Service, Ingress, and HPA, templated via `values.yaml`
- **`.github/workflows/build-and-push.yml`** — GitHub Actions workflow that triggers on every push to `gitops`; builds the Docker image and pushes it to ECR (tagged with the commit SHA and `latest`)
- **`argocd-application.yaml`** — an Argo CD Application resource that watches the `gitops` branch's `techchallenge2-chart` folder, with automated sync and self-healing enabled

Argo CD is installed in the `argocd` namespace on the same EKS cluster, and continuously reconciles the live cluster state to match what's defined in the Helm chart on GitHub — any future push to `gitops` is automatically built (via GitHub Actions) and deployed (via Argo CD), with no manual `kubectl` or `helm` commands required.

## Repository Structure
techchallenge2/
├── app.py                       # Flask "Hello, World!" app
├── Dockerfile
├── requirements.txt
├── Jenkinsfile                  # Jenkins CI/CD pipeline (main branch)
├── terraform/                   # EKS cluster infrastructure as code
│   ├── provider.tf
│   ├── vpc.tf
│   └── eks.tf
├── k8s/                         # Kubernetes manifests (main branch deployment)
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── hpa.yaml
│   └── ingress.yaml
├── techchallenge2-chart/        # Helm chart (gitops branch)
├── .github/workflows/           # GitHub Actions CI (gitops branch)
└── argocd-application.yaml      # Argo CD Application manifest (gitops branch)
