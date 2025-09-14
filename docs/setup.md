# Setup Guide

## Configuración Inicial del Proyecto GitOps

### Requisitos Previos
- Docker instalado
- Kubernetes cluster (local o remoto)
- kubectl configurado
- GitHub account
- ArgoCD instalado en el cluster

### 1. Configuración de Variables de Entorno

Crea un archivo `.env` en la raíz del proyecto:

```bash
# Docker Registry
DOCKER_REGISTRY=ghcr.io
DOCKER_USERNAME=tu-usuario-github
DOCKER_PASSWORD=tu-token-github

# Kubernetes
KUBECONFIG_PATH=~/.kube/config
NAMESPACE=gitops-demo

# ArgoCD
ARGOCD_SERVER=localhost:8080
ARGOCD_USERNAME=admin
```

### 2. Configuración de GitHub Actions

Configura los siguientes secrets en tu repositorio de GitHub:

```yaml
Secrets requeridos:
- DOCKER_REGISTRY: ghcr.io
- DOCKER_USERNAME: tu-usuario
- DOCKER_TOKEN: token-con-permisos-write:packages
```

### 3. Configuración del Cluster Kubernetes

```bash
# Crear namespace
kubectl create namespace gitops-demo

# Aplicar manifiestos
kubectl apply -f Kubernetes/ -n gitops-demo
```

### 4. Configuración de ArgoCD

```bash
# Instalar ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Obtener password inicial
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port forward para acceso local
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### 5. Crear Aplicación en ArgoCD

```bash
argocd app create gitops-demo \
  --repo https://github.com/Portfolio-jaime/GitOps.git \
  --path Kubernetes \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace gitops-demo \
  --sync-policy automated
```

### Verificación

```bash
# Verificar estado de la aplicación
argocd app get gitops-demo

# Verificar pods
kubectl get pods -n gitops-demo

# Verificar servicios
kubectl get svc -n gitops-demo
```