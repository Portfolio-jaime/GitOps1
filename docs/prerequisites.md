# Prerequisites

Requirements and setup needed before running the GitOps demo project.

## 🖥️ System Requirements

### Hardware Requirements
- **CPU**: 2+ cores (4+ recommended)
- **RAM**: 4GB minimum (8GB recommended)
- **Storage**: 10GB available space
- **Network**: Internet connection for downloading images

### Operating System Support
- **Linux**: Ubuntu 18.04+, CentOS 7+, Debian 9+
- **macOS**: 10.14 Mojave or later
- **Windows**: Windows 10 with WSL2

## 🛠️ Required Tools

### Essential Tools
| Tool | Version | Purpose | Installation |
|------|---------|---------|--------------|
| **Docker** | 20.0+ | Container runtime | [Docker Install](https://docs.docker.com/get-docker/) |
| **kubectl** | 1.25+ | Kubernetes CLI | [kubectl Install](https://kubernetes.io/docs/tasks/tools/install-kubectl/) |
| **Git** | 2.20+ | Version control | [Git Install](https://git-scm.com/downloads) |

### Kubernetes Cluster
Choose one of the following options:

#### Option 1: Minikube (Recommended for Demo)
```bash
# Install Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start cluster
minikube start --memory=4096 --cpus=2
minikube status
```

#### Option 2: Docker Desktop
```bash
# Enable Kubernetes in Docker Desktop
# Settings → Kubernetes → Enable Kubernetes
kubectl cluster-info
```

#### Option 3: Cloud Kubernetes
- **EKS** (AWS): [EKS Getting Started](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html)
- **GKE** (Google): [GKE Quickstart](https://cloud.google.com/kubernetes-engine/docs/quickstart)
- **AKS** (Azure): [AKS Quickstart](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough)

### CI/CD Tools

#### GitHub Actions (Included)
- GitHub account with repository access
- GitHub Actions enabled (default for public repos)

#### ArgoCD
```bash
# Install ArgoCD using kubectl
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Or using Helm (recommended)
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd --create-namespace
```

## 🔧 Installation Instructions

### 1. Install Docker

#### Ubuntu/Debian
```bash
# Update package index
sudo apt update

# Install dependencies
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release

# Add Docker GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Verify installation
docker --version
docker run hello-world
```

#### macOS
```bash
# Using Homebrew
brew install --cask docker

# Or download from Docker website
# https://docs.docker.com/desktop/mac/install/
```

#### Windows (WSL2)
```bash
# Install Docker Desktop for Windows
# Enable WSL2 backend in Docker Desktop settings
# Install Docker in WSL2:
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
```

### 2. Install kubectl

#### Using curl
```bash
# Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# macOS
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

#### Using Package Managers
```bash
# macOS with Homebrew
brew install kubectl

# Ubuntu/Debian
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubectl
```

#### Verify Installation
```bash
kubectl version --client
kubectl cluster-info
```

### 3. Install Git
```bash
# Ubuntu/Debian
sudo apt install -y git

# CentOS/RHEL
sudo yum install -y git

# macOS
brew install git

# Verify
git --version
```

## 🚀 Kubernetes Cluster Setup

### Minikube Setup (Recommended)
```bash
# Install Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start cluster with adequate resources
minikube start \
  --memory=4096 \
  --cpus=2 \
  --disk-size=20gb \
  --driver=docker

# Enable useful addons
minikube addons enable ingress
minikube addons enable dashboard
minikube addons enable metrics-server

# Verify cluster
kubectl get nodes
kubectl get pods -A
```

### Alternative: Kind (Kubernetes in Docker)
```bash
# Install Kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Create cluster
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
EOF

# Verify
kubectl cluster-info --context kind-kind
```

## 🎯 ArgoCD Installation

### Method 1: Official Manifests
```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods to be ready
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

### Method 2: Helm (Recommended)
```bash
# Add Helm repository
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Install ArgoCD with custom values
cat > argocd-values.yaml <<EOF
server:
  service:
    type: LoadBalancer
  config:
    repositories: |
      - url: https://github.com/Portfolio-jaime/GitOps.git
        type: git
EOF

# Install
helm install argocd argo/argo-cd \
  -n argocd \
  --create-namespace \
  -f argocd-values.yaml \
  --wait

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

### Access ArgoCD UI
```bash
# Port forward to access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# Or with Minikube
minikube service argocd-server -n argocd

# Access at: https://localhost:8080
# Username: admin
# Password: (from previous step)
```

## 🔐 Container Registry Setup

### Docker Hub (Free Option)
```bash
# Create account at https://hub.docker.com
# Login from command line
docker login

# Test push (optional)
docker tag hello-world yourusername/hello-world
docker push yourusername/hello-world
```

### GitHub Container Registry (Recommended)
```bash
# Create Personal Access Token with packages:write scope
# https://github.com/settings/tokens

# Login to GitHub Container Registry
echo $GITHUB_TOKEN | docker login ghcr.io -u yourusername --password-stdin

# Test access
docker pull ghcr.io/yourusername/test:latest
```

### Alternative: Local Registry
```bash
# Start local registry for testing
docker run -d -p 5000:5000 --restart=always --name registry registry:2

# Configure Docker daemon to use insecure registry
# Add to /etc/docker/daemon.json:
{
  "insecure-registries": ["localhost:5000"]
}

# Restart Docker
sudo systemctl restart docker
```

## 🔑 GitHub Setup

### Repository Access
1. **Fork or Clone**: Fork the GitOps repository or create your own
2. **Access Tokens**: Create GitHub Personal Access Token
   - Go to GitHub Settings → Developer settings → Personal access tokens
   - Generate token with `repo` and `workflow` scopes

### GitHub Actions Secrets
Configure the following secrets in your repository:
- `DOCKER_USERNAME`: Docker Hub username
- `DOCKER_PASSWORD`: Docker Hub password or token
- `KUBE_CONFIG`: Base64 encoded kubeconfig (for external clusters)

```bash
# Encode kubeconfig for GitHub secrets
cat ~/.kube/config | base64 -w 0
```

## 🧪 Verification

### Pre-flight Checklist
```bash
# 1. Docker is running
docker --version
docker ps

# 2. Kubernetes cluster is accessible
kubectl cluster-info
kubectl get nodes

# 3. ArgoCD is installed and running
kubectl get pods -n argocd
kubectl get svc -n argocd

# 4. Git is configured
git --version
git config --list

# 5. Container registry access
docker login
```

### Test GitOps Workflow
```bash
# 1. Clone the repository
git clone https://github.com/Portfolio-jaime/GitOps.git
cd GitOps

# 2. Build and test Docker image
docker build -t test-gitops Docker/
docker run -p 8080:80 test-gitops &
curl http://localhost:8080
docker stop $(docker ps -q --filter ancestor=test-gitops)

# 3. Test Kubernetes deployment
kubectl apply -f Kubernetes/
kubectl get pods
kubectl get svc

# 4. Cleanup test deployment
kubectl delete -f Kubernetes/
```

## ⚠️ Common Issues

### Docker Permission Issues
```bash
# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Or run with sudo (not recommended)
sudo docker ps
```

### Kubernetes Connection Issues
```bash
# Check kubeconfig
kubectl config view
kubectl config current-context

# For Minikube
minikube status
minikube start

# Verify cluster access
kubectl auth can-i get pods
```

### ArgoCD Access Issues
```bash
# Check ArgoCD pods
kubectl get pods -n argocd

# Reset admin password
kubectl -n argocd patch secret argocd-secret \
  -p '{"stringData": {"admin.password": "$2a$10$rRyBsGSHK6.uc8fntPwVIuLVHgsAhAX7TcdrqW/RADU0uh7CaChLa","admin.passwordMtime": "'$(date +%FT%T%Z)'"}}'

# New password is: password
```

## 📚 Additional Resources

### Documentation
- [Docker Documentation](https://docs.docker.com/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)

### Tutorials
- [Kubernetes Basics](https://kubernetes.io/docs/tutorials/kubernetes-basics/)
- [ArgoCD Getting Started](https://argo-cd.readthedocs.io/en/stable/getting_started/)
- [GitHub Actions Quickstart](https://docs.github.com/en/actions/quickstart)

### Tools
- [Minikube](https://minikube.sigs.k8s.io/)
- [Kind](https://kind.sigs.k8s.io/)
- [Docker Desktop](https://www.docker.com/products/docker-desktop)
- [Helm](https://helm.sh/)

---

**✅ Once you've completed these prerequisites, you're ready to proceed with the [Setup Guide](setup.md)!**