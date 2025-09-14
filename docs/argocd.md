# ArgoCD Configuration

## Instalación y Configuración

### Instalación de ArgoCD
```bash
# Crear namespace
kubectl create namespace argocd

# Instalar ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Verificar instalación
kubectl get pods -n argocd

# Esperar que todos los pods estén listos
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
```

### Acceso a ArgoCD UI
```bash
# Obtener password inicial del admin
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

# Port forward para acceso local
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Acceder en el navegador: https://localhost:8080
# Usuario: admin
# Password: [obtenido del comando anterior]
```

### Configuración de CLI
```bash
# Instalar ArgoCD CLI
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

# Login via CLI
argocd login localhost:8080

# Cambiar password (opcional)
argocd account update-password
```

## Configuración de Aplicación GitOps

### Manifest de Aplicación ArgoCD
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitops-demo
  namespace: argocd
  labels:
    app.kubernetes.io/name: gitops-demo
    app.kubernetes.io/part-of: gitops-portfolio
  annotations:
    argocd.argoproj.io/sync-wave: "1"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/Portfolio-jaime/GitOps.git
    targetRevision: HEAD
    path: Kubernetes
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: gitops-demo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  revisionHistoryLimit: 10
  info:
  - name: 'GitOps Demo Application'
    value: 'Aplicación de demostración para pipeline GitOps con Kubernetes y ArgoCD'
```

### Creación via CLI
```bash
# Crear aplicación
argocd app create gitops-demo \
  --repo https://github.com/Portfolio-jaime/GitOps.git \
  --path Kubernetes \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace gitops-demo \
  --sync-policy automated \
  --auto-prune \
  --self-heal \
  --sync-option CreateNamespace=true

# Verificar aplicación
argocd app get gitops-demo

# Sincronizar manualmente (si es necesario)
argocd app sync gitops-demo

# Ver logs de sincronización
argocd app logs gitops-demo
```

## Configuración de Project

### ArgoCD Project
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: gitops-portfolio
  namespace: argocd
  labels:
    app.kubernetes.io/name: gitops-portfolio
    app.kubernetes.io/part-of: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  description: 'Portfolio de aplicaciones GitOps'
  
  # Repositorios permitidos
  sourceRepos:
  - 'https://github.com/Portfolio-jaime/*'
  - 'https://charts.helm.sh/stable'
  - 'https://kubernetes-charts.storage.googleapis.com'
  
  # Destinos permitidos
  destinations:
  - namespace: 'gitops-*'
    server: https://kubernetes.default.svc
  - namespace: 'portfolio-*'
    server: https://kubernetes.default.svc
  
  # Recursos permitidos por cluster
  clusterResourceWhitelist:
  - group: ''
    kind: Namespace
  - group: 'rbac.authorization.k8s.io'
    kind: ClusterRole
  - group: 'rbac.authorization.k8s.io'
    kind: ClusterRoleBinding
  
  # Recursos permitidos por namespace
  namespaceResourceWhitelist:
  - group: ''
    kind: ConfigMap
  - group: ''
    kind: Secret
  - group: ''
    kind: Service
  - group: ''
    kind: ServiceAccount
  - group: 'apps'
    kind: Deployment
  - group: 'apps'
    kind: ReplicaSet
  - group: 'networking.k8s.io'
    kind: Ingress
  - group: 'networking.k8s.io'
    kind: NetworkPolicy
  - group: 'autoscaling'
    kind: HorizontalPodAutoscaler
  - group: 'policy'
    kind: PodDisruptionBudget
  
  # Roles y políticas
  roles:
  - name: developer
    description: 'Desarrolladores del portfolio'
    policies:
    - p, proj:gitops-portfolio:developer, applications, get, gitops-portfolio/*, allow
    - p, proj:gitops-portfolio:developer, applications, sync, gitops-portfolio/*, allow
    - p, proj:gitops-portfolio:developer, applications, action/*, gitops-portfolio/*, allow
    - p, proj:gitops-portfolio:developer, repositories, get, *, allow
    groups:
    - gitops-portfolio:developers
  
  - name: admin
    description: 'Administradores del portfolio'
    policies:
    - p, proj:gitops-portfolio:admin, applications, *, gitops-portfolio/*, allow
    - p, proj:gitops-portfolio:admin, repositories, *, *, allow
    - p, proj:gitops-portfolio:admin, clusters, *, *, allow
    groups:
    - gitops-portfolio:admins

  # Configuración de sincronización
  syncWindows:
  - kind: allow
    schedule: '* * * * *'
    duration: 24h
    applications:
    - gitops-demo
    manualSync: true
  - kind: deny
    schedule: '0 2 * * 1-5'
    duration: 1h
    applications:
    - '*'
    manualSync: false
    timeZone: 'Europe/London'
```

## Configuración Avanzada

### Repository Configuration
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitops-repo-secret
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
type: Opaque
stringData:
  type: git
  url: https://github.com/Portfolio-jaime/GitOps.git
  username: Portfolio-jaime
  password: ghp_xxxxxxxxxxxxxxxxxxxx
  tlsClientCertData: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
  tlsClientCertKey: |
    -----BEGIN RSA PRIVATE KEY-----
    ...
    -----END RSA PRIVATE KEY-----
```

### Webhook Configuration
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  name: argocd
  namespace: argocd
spec:
  server:
    insecure: true
    service:
      type: LoadBalancer
    ingress:
      enabled: true
      ingressClassName: nginx
      annotations:
        nginx.ingress.kubernetes.io/ssl-redirect: "true"
        nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
        cert-manager.io/cluster-issuer: letsencrypt-prod
      hosts:
      - argocd.example.com
      tls:
      - secretName: argocd-server-tls
        hosts:
        - argocd.example.com
    
  # Configuración de repositorios
  repositories: |
    - type: git
      url: https://github.com/Portfolio-jaime/GitOps.git
      name: gitops-portfolio
    
  # Configuración de credenciales
  repositoryCredentials: |
    - url: https://github.com/Portfolio-jaime
      passwordSecret:
        name: gitops-repo-secret
        key: password
      usernameSecret:
        name: gitops-repo-secret
        key: username
  
  # Configuración RBAC
  rbac:
    defaultPolicy: 'role:readonly'
    policy: |
      p, role:admin, applications, *, */*, allow
      p, role:admin, clusters, *, *, allow
      p, role:admin, repositories, *, *, allow
      
      p, role:developer, applications, get, */*, allow
      p, role:developer, applications, sync, */*, allow
      p, role:developer, applications, action/*, */*, allow
      
      g, argocd-admins, role:admin
      g, gitops-developers, role:developer
    
    scopes: '[groups]'

  # Configuración de recursos
  resourceCustomizations: |
    networking.k8s.io/Ingress:
      health.lua: |
        hs = {}
        hs.status = "Healthy"
        return hs
```

## Monitoreo y Alertas

### ServiceMonitor para Prometheus
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-metrics
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-metrics
    app.kubernetes.io/part-of: argocd
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-metrics
      app.kubernetes.io/part-of: argocd
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

### Grafana Dashboard
```json
{
  "dashboard": {
    "id": null,
    "title": "ArgoCD GitOps Dashboard",
    "tags": ["argocd", "gitops"],
    "timezone": "browser",
    "panels": [
      {
        "title": "Application Health",
        "type": "stat",
        "targets": [
          {
            "expr": "argocd_app_health_status{application=\"gitops-demo\"}",
            "legendFormat": "Health Status"
          }
        ]
      },
      {
        "title": "Sync Status",
        "type": "stat",
        "targets": [
          {
            "expr": "argocd_app_sync_total{application=\"gitops-demo\"}",
            "legendFormat": "Sync Count"
          }
        ]
      },
      {
        "title": "Repository Activity",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(argocd_git_request_total[5m])",
            "legendFormat": "Git Requests/sec"
          }
        ]
      }
    ]
  }
}
```

## Scripts de Utilidad

### setup-argocd.sh
```bash
#!/bin/bash
set -e

echo "Installing ArgoCD..."

# Install ArgoCD
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

echo "Waiting for ArgoCD to be ready..."
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s

# Get admin password
ARGOCD_PASSWORD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)

echo "ArgoCD Installation Complete!"
echo "================================"
echo "Admin Password: $ARGOCD_PASSWORD"
echo ""
echo "To access ArgoCD UI:"
echo "kubectl port-forward svc/argocd-server -n argocd 8080:443"
echo "Then visit: https://localhost:8080"
echo ""
echo "To create the GitOps application:"
echo "./create-app.sh"
```

### create-app.sh
```bash
#!/bin/bash
set -e

APP_NAME="gitops-demo"
REPO_URL="https://github.com/Portfolio-jaime/GitOps.git"
PATH_IN_REPO="Kubernetes"
NAMESPACE="gitops-demo"

echo "Creating ArgoCD application: $APP_NAME"

# Create application
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: $APP_NAME
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: $REPO_URL
    targetRevision: HEAD
    path: $PATH_IN_REPO
  destination:
    server: https://kubernetes.default.svc
    namespace: $NAMESPACE
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF

echo "Application '$APP_NAME' created successfully!"
echo ""
echo "To check status:"
echo "kubectl get application $APP_NAME -n argocd"
echo ""
echo "To view in UI:"
echo "kubectl port-forward svc/argocd-server -n argocd 8080:443"
```

### sync-app.sh
```bash
#!/bin/bash

APP_NAME=${1:-gitops-demo}

echo "Syncing ArgoCD application: $APP_NAME"

# Check if application exists
if ! kubectl get application $APP_NAME -n argocd >/dev/null 2>&1; then
  echo "Error: Application '$APP_NAME' not found"
  exit 1
fi

# Trigger sync
kubectl patch application $APP_NAME -n argocd \
  --type='merge' \
  -p='{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'

echo "Sync triggered for application '$APP_NAME'"
echo "Check status with: kubectl get application $APP_NAME -n argocd -w"
```