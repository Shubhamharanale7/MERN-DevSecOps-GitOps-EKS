# MERN Stack — DevSecOps + GitOps on AWS EKS

A production-grade DevSecOps and GitOps pipeline for a three-tier MERN stack application deployed on AWS EKS using Jenkins CI, ArgoCD CD, SonarQube, OWASP, Trivy, and Helm-based monitoring.

## Architecture

![Architecture](./Devops-Mega-Project-Jenkins-ArgoCD-EKS/Assets/architectures.png)

## CI/CD Flow

![CI/CD Flow](./Devops-Mega-Project-Jenkins-ArgoCD-EKS/Assets/flow.png)

```
Developer → GitHub
               └── Jenkins CI Job
                     ├── OWASP Dependency Check
                     ├── SonarQube Analysis
                     ├── Trivy Filesystem Scan
                     ├── Docker Build & Push
                     └── Trigger Jenkins CD Job
                               └── Update Docker Image Version → GitHub
                                         └── ArgoCD Auto Sync
                                                   └── Deploy to AWS EKS
                                                             └── Prometheus + Grafana Monitoring
                                                                       └── Email Notification
```

## Tech Stack

| Category | Tools |
|----------|-------|
| Source Control | GitHub |
| CI | Jenkins |
| Code Quality | SonarQube |
| Security Scanning | OWASP Dependency Check, Trivy |
| Containerization | Docker |
| CD / GitOps | ArgoCD |
| Orchestration | AWS EKS (Kubernetes) |
| Monitoring | Prometheus + Grafana (Helm) |
| Caching | Redis |
| Notifications | Gmail SMTP |

## Pipeline Stages

### CI Pipeline (Jenkins)
- ✅ Code Checkout from GitHub
- ✅ Trivy Filesystem Scan
- ✅ OWASP Dependency Check
- ✅ SonarQube Code Analysis + Quality Gate
- ✅ Docker Image Build
- ✅ Docker Push to DockerHub
- ✅ Trigger CD Pipeline

### CD Pipeline (GitOps with ArgoCD)
- ✅ Update Kubernetes Manifests with new image version
- ✅ Push changes to GitHub
- ✅ ArgoCD Auto Sync
- ✅ Rolling Deployment to AWS EKS
- ✅ Zero Downtime Deployment
- ✅ Email Notification

## Infrastructure Setup

### AWS EC2 Instances Required

| Server | Type | Storage | Purpose |
|--------|------|---------|---------|
| Master | t2.large | 30GB | Jenkins Master, eksctl, EKS cluster |
| Jenkins Worker | t2.large | 29GB | Build agent, Docker, Trivy |
| SonarQube | t2.medium | 25GB | Code quality analysis |

### 1. Install Docker
```bash
sudo apt-get update
sudo apt-get install docker.io -y
sudo usermod -aG docker ubuntu && newgrp docker
```

### 2. Install Jenkins (Master)
```bash
sudo apt update -y
sudo apt install fontconfig openjdk-21-jre -y
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update -y
sudo apt install jenkins -y
```

### 3. Create EKS Cluster
```bash
# Install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Create cluster
eksctl create cluster --name=wanderlust \
                      --region=us-east-2 \
                      --version=1.34 \
                      --without-nodegroup

# Associate IAM OIDC Provider
eksctl utils associate-iam-oidc-provider \
  --region us-east-2 \
  --cluster wanderlust \
  --approve

# Create Nodegroup
eksctl create nodegroup --cluster=wanderlust \
                        --region=us-east-2 \
                        --name=wanderlust \
                        --node-type=t3.medium \
                        --nodes=2 \
                        --nodes-min=2 \
                        --nodes-max=3 \
                        --node-volume-size=30 \
                        --ssh-access \
                        --ssh-public-key=eks-nodegroup-key
```

### 4. Install ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

# Get initial password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

### 5. Install Trivy (Jenkins Worker)
```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update -y
sudo apt-get install trivy -y
```

## Jenkins Plugins Required
- OWASP Dependency Check
- SonarQube Scanner
- Docker & Docker Pipeline
- Pipeline: Stage View
- Kubernetes & Kubernetes CLI

## Jenkins Credentials Required

| ID | Type | Value |
|----|------|-------|
| `sonar-token` | Secret text | SonarQube token |
| `docker` | Username/Password | DockerHub credentials |
| `github` | Username/Password | GitHub PAT |
| `k8s` | Secret text | K8s service account token |
| `mail-cred` | Username/Password | Gmail app password |

## Monitoring with Prometheus + Grafana (Helm)

```bash
# Install Helm
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh && ./get_helm.sh

# Add repos
helm repo add stable https://charts.helm.sh/stable
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Install
kubectl create namespace prometheus
helm install stable prometheus-community/kube-prometheus-stack -n prometheus

# Expose Grafana
kubectl edit svc stable-grafana -n prometheus  # Change ClusterIP to NodePort

# Get Grafana password
kubectl get secret --namespace prometheus stable-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode; echo
```

## Access the Application

```
Frontend: http://<worker-public-ip>:31000
Backend:  http://<worker-public-ip>:31100
```

## Cleanup

```bash
eksctl delete cluster --name=wanderlust --region=us-east-2
```

---

**Built by [Shubham Haranale](https://github.com/Shubhamharanale7)**

[![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/Shubhamharanale7)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-%230077B5.svg?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/shubhamharanale7)
