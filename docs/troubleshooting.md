# Troubleshooting Guide

## Problemas Comunes de Despliegue

### 1. Imagen Docker no se actualiza

#### Síntomas
- Los pods siguen ejecutando la versión anterior de la aplicación
- ArgoCD muestra estado "Synced" pero la aplicación no cambió

#### Diagnóstico
```bash
# Verificar la imagen actual en el deployment
kubectl describe deployment gitops-demo -n gitops-demo | grep Image

# Verificar la imagen en los pods
kubectl get pods -n gitops-demo -o jsonpath='{.items[*].spec.containers[*].image}'

# Verificar el tag de la imagen en el registry
docker manifest inspect ghcr.io/portfolio-jaime/gitops:latest
```

#### Soluciones
```bash
# 1. Forzar recreación de pods
kubectl rollout restart deployment/gitops-demo -n gitops-demo

# 2. Usar imagePullPolicy: Always
kubectl patch deployment gitops-demo -n gitops-demo -p '{"spec":{"template":{"spec":{"containers":[{"name":"webapp","imagePullPolicy":"Always"}]}}}}'

# 3. Verificar que el tag de imagen sea único
# En el workflow de GitHub Actions, usar SHA en lugar de 'latest'
sed -i "s|image: .*|image: ghcr.io/portfolio-jaime/gitops:${GITHUB_SHA}|" Kubernetes/deployment.yaml
```

### 2. ArgoCD no sincroniza automáticamente

#### Síntomas
- Los cambios en Git no se reflejan en el cluster
- ArgoCD muestra "OutOfSync" pero no sincroniza automáticamente

#### Diagnóstico
```bash
# Verificar el estado de la aplicación
argocd app get gitops-demo

# Verificar los webhooks de GitHub
curl -s -H "Authorization: token $GITHUB_TOKEN" \
  https://api.github.com/repos/Portfolio-jaime/GitOps/hooks

# Verificar logs de ArgoCD
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller
```

#### Soluciones
```bash
# 1. Verificar configuración de sync policy
kubectl patch application gitops-demo -n argocd --type='merge' -p='{
  "spec": {
    "syncPolicy": {
      "automated": {
        "prune": true,
        "selfHeal": true
      }
    }
  }
}'

# 2. Sincronización manual
argocd app sync gitops-demo

# 3. Refresh manual de la aplicación
argocd app get gitops-demo --refresh

# 4. Verificar permisos del repositorio
argocd repo get https://github.com/Portfolio-jaime/GitOps.git
```

### 3. Problemas de Red y Conectividad

#### Síntomas
- Pods no pueden conectarse entre sí
- Aplicación no responde desde el exterior
- Timeouts en health checks

#### Diagnóstico
```bash
# Verificar estado de los pods
kubectl get pods -n gitops-demo -o wide

# Verificar servicios y endpoints
kubectl get svc,endpoints -n gitops-demo

# Probar conectividad desde dentro del cluster
kubectl exec -it deployment/gitops-demo -n gitops-demo -- wget -qO- http://localhost:8080/health

# Verificar Network Policies
kubectl get networkpolicy -n gitops-demo
```

#### Soluciones
```bash
# 1. Verificar configuración del Service
kubectl describe service gitops-demo -n gitops-demo

# 2. Verificar selectors del Service
kubectl get pods -n gitops-demo --show-labels
kubectl describe service gitops-demo -n gitops-demo | grep Selector

# 3. Probar port-forward directo
kubectl port-forward deployment/gitops-demo 8080:8080 -n gitops-demo

# 4. Verificar Ingress (si está configurado)
kubectl describe ingress gitops-demo -n gitops-demo

# 5. Verificar DNS interno
kubectl exec -it deployment/gitops-demo -n gitops-demo -- nslookup gitops-demo.gitops-demo.svc.cluster.local
```

### 4. Problemas de Recursos

#### Síntomas
- Pods en estado "Pending" o "CrashLoopBackOff"
- Pods reiniciándose frecuentemente
- Performance degradada de la aplicación

#### Diagnóstico
```bash
# Verificar recursos del cluster
kubectl top nodes
kubectl top pods -n gitops-demo

# Verificar eventos del namespace
kubectl get events -n gitops-demo --sort-by=.metadata.creationTimestamp

# Verificar límites de recursos
kubectl describe deployment gitops-demo -n gitops-demo | grep -A 10 "Limits\|Requests"

# Verificar estado de los pods
kubectl describe pods -n gitops-demo
```

#### Soluciones
```bash
# 1. Ajustar requests y limits de CPU/memoria
kubectl patch deployment gitops-demo -n gitops-demo -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "webapp",
          "resources": {
            "requests": {
              "cpu": "100m",
              "memory": "128Mi"
            },
            "limits": {
              "cpu": "500m",
              "memory": "256Mi"
            }
          }
        }]
      }
    }
  }
}'

# 2. Configurar HorizontalPodAutoscaler
kubectl apply -f - <<EOF
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: gitops-demo-hpa
  namespace: gitops-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: gitops-demo
  minReplicas: 2
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
EOF

# 3. Verificar quotas del namespace
kubectl describe quota -n gitops-demo
kubectl describe limitrange -n gitops-demo
```

### 5. Problemas de Certificados TLS

#### Síntomas
- Errores SSL/TLS al acceder a la aplicación
- Certificados expirados o no válidos
- Navegador muestra advertencias de seguridad

#### Diagnóstico
```bash
# Verificar certificados en el Ingress
kubectl describe ingress gitops-demo -n gitops-demo

# Verificar secretos TLS
kubectl get secrets -n gitops-demo | grep tls
kubectl describe secret gitops-demo-tls -n gitops-demo

# Verificar cert-manager (si está instalado)
kubectl get certificates -n gitops-demo
kubectl describe certificate gitops-demo-tls -n gitops-demo

# Probar conectividad TLS
openssl s_client -connect gitops-demo.example.com:443 -servername gitops-demo.example.com
```

#### Soluciones
```bash
# 1. Regenerar certificado con cert-manager
kubectl delete certificate gitops-demo-tls -n gitops-demo
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: gitops-demo-tls
  namespace: gitops-demo
spec:
  secretName: gitops-demo-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - gitops-demo.example.com
EOF

# 2. Verificar el ClusterIssuer
kubectl describe clusterissuer letsencrypt-prod

# 3. Verificar logs de cert-manager
kubectl logs -n cert-manager -l app=cert-manager
```

## Problemas de GitHub Actions

### 1. Pipeline falla en el build de Docker

#### Síntomas
- Error "no space left on device"
- Timeout en docker build
- Error de autenticación con el registry

#### Diagnóstico y Soluciones
```bash
# 1. Limpiar espacio en disco
- name: Clean up Docker
  run: |
    docker system prune -a -f
    docker volume prune -f
    df -h

# 2. Usar BuildKit con caché
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3
  with:
    buildkitd-flags: --debug

- name: Build with cache
  uses: docker/build-push-action@v5
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max

# 3. Verificar autenticación
- name: Log in to Container Registry
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
```

### 2. Tests fallan intermitentemente

#### Síntomas
- Tests pasan localmente pero fallan en CI
- Errores de timeout o race conditions
- Dependencias externas no disponibles

#### Soluciones
```bash
# 1. Agregar retry logic
- name: Run tests with retry
  uses: nick-invision/retry@v2
  with:
    timeout_minutes: 10
    max_attempts: 3
    command: npm test

# 2. Usar servicios containerizados
services:
  redis:
    image: redis:alpine
    ports:
      - 6379:6379
  
  postgres:
    image: postgres:13
    env:
      POSTGRES_PASSWORD: test
    ports:
      - 5432:5432

# 3. Configurar timeouts adecuados
- name: Run tests
  run: npm test
  env:
    TEST_TIMEOUT: 30000
    NODE_ENV: test
```

## Problemas de Monitoreo

### 1. Métricas no disponibles

#### Diagnóstico
```bash
# Verificar ServiceMonitor
kubectl get servicemonitor -n gitops-demo
kubectl describe servicemonitor gitops-demo -n gitops-demo

# Verificar endpoints de métricas
kubectl get endpoints -n gitops-demo
curl -s http://<pod-ip>:8080/metrics

# Verificar configuración de Prometheus
kubectl get prometheus -n monitoring
kubectl logs -n monitoring -l app=prometheus
```

#### Soluciones
```bash
# 1. Verificar anotaciones de Prometheus
kubectl patch deployment gitops-demo -n gitops-demo -p '{
  "spec": {
    "template": {
      "metadata": {
        "annotations": {
          "prometheus.io/scrape": "true",
          "prometheus.io/port": "8080",
          "prometheus.io/path": "/metrics"
        }
      }
    }
  }
}'

# 2. Crear ServiceMonitor manualmente
kubectl apply -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: gitops-demo
  namespace: gitops-demo
spec:
  selector:
    matchLabels:
      app: gitops-demo
  endpoints:
  - port: http
    path: /metrics
EOF
```

## Scripts de Debugging

### debug-deployment.sh
```bash
#!/bin/bash

NAMESPACE=${1:-gitops-demo}
DEPLOYMENT=${2:-gitops-demo}

echo "=== Debugging deployment $DEPLOYMENT in namespace $NAMESPACE ==="

echo "1. Deployment status:"
kubectl get deployment $DEPLOYMENT -n $NAMESPACE -o wide

echo -e "\n2. Deployment events:"
kubectl describe deployment $DEPLOYMENT -n $NAMESPACE | tail -20

echo -e "\n3. Pod status:"
kubectl get pods -l app=$DEPLOYMENT -n $NAMESPACE -o wide

echo -e "\n4. Pod logs (last 50 lines):"
kubectl logs -l app=$DEPLOYMENT -n $NAMESPACE --tail=50

echo -e "\n5. Service and endpoints:"
kubectl get svc,endpoints -l app=$DEPLOYMENT -n $NAMESPACE

echo -e "\n6. Recent events:"
kubectl get events -n $NAMESPACE --sort-by=.metadata.creationTimestamp | tail -10

echo -e "\n7. Resource usage:"
kubectl top pods -n $NAMESPACE 2>/dev/null || echo "Metrics server not available"
```

### check-argocd.sh
```bash
#!/bin/bash

APP_NAME=${1:-gitops-demo}

echo "=== Checking ArgoCD application $APP_NAME ==="

echo "1. Application status:"
argocd app get $APP_NAME

echo -e "\n2. Application sync status:"
argocd app sync $APP_NAME --dry-run

echo -e "\n3. Application diff:"
argocd app diff $APP_NAME

echo -e "\n4. Application logs:"
argocd app logs $APP_NAME --tail=20

echo -e "\n5. Repository connection:"
argocd repo get $(argocd app get $APP_NAME -o json | jq -r '.spec.source.repoURL')
```

### health-check.sh
```bash
#!/bin/bash

NAMESPACE=${1:-gitops-demo}
SERVICE=${2:-gitops-demo}

echo "=== Health check for $SERVICE in $NAMESPACE ==="

# Port forward en background
kubectl port-forward svc/$SERVICE 8080:80 -n $NAMESPACE &
PF_PID=$!

# Esperar que el port forward esté listo
sleep 3

echo "1. Health endpoint:"
curl -s http://localhost:8080/health || echo "Health endpoint not responding"

echo -e "\n2. Basic connectivity:"
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080 || echo "Service not responding"

echo -e "\n3. Response time:"
curl -s -o /dev/null -w "%{time_total}s\n" http://localhost:8080 || echo "Unable to measure response time"

# Limpiar port forward
kill $PF_PID 2>/dev/null

echo -e "\n4. DNS resolution:"
kubectl exec -it deployment/$SERVICE -n $NAMESPACE -- nslookup $SERVICE.$NAMESPACE.svc.cluster.local 2>/dev/null || echo "DNS resolution failed"
```

## Logs y Observabilidad

### Centralización de Logs
```bash
# Configurar Fluent Bit para recolección de logs
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: gitops-demo
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:latest
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: config
          mountPath: /fluent-bit/etc
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: config
        configMap:
          name: fluent-bit-config
EOF
```

### Structured Logging
```javascript
// Ejemplo de logging estructurado en la aplicación
const logger = require('winston');

logger.configure({
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: {
    service: 'gitops-demo',
    namespace: process.env.NAMESPACE || 'gitops-demo',
    pod: process.env.HOSTNAME
  },
  transports: [
    new winston.transports.Console()
  ]
});

// Uso en el código
logger.info('Application started', {
  port: process.env.PORT || 8080,
  nodeVersion: process.version
});

logger.error('Database connection failed', {
  error: err.message,
  stack: err.stack,
  connectionString: 'postgres://...'
});
```

### Alerting Rules
```yaml
# Prometheus alerting rules
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: gitops-demo-alerts
  namespace: gitops-demo
spec:
  groups:
  - name: gitops-demo
    rules:
    - alert: GitOpsDemoDown
      expr: up{job="gitops-demo"} == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "GitOps Demo application is down"
        description: "GitOps Demo application has been down for more than 5 minutes"
    
    - alert: GitOpsDemoHighErrorRate
      expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.1
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "High error rate detected"
        description: "Error rate is {{ $value }} errors per second"
```