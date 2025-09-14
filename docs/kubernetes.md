# Kubernetes Configuration

## Manifiestos Base

### Namespace
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gitops-demo
  labels:
    name: gitops-demo
    app.kubernetes.io/name: gitops-demo
    app.kubernetes.io/part-of: gitops-demo
  annotations:
    description: "Namespace for GitOps demo application"
```

### Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitops-demo
  namespace: gitops-demo
  labels:
    app: gitops-demo
    app.kubernetes.io/name: gitops-demo
    app.kubernetes.io/instance: gitops-demo
    app.kubernetes.io/version: "1.0"
    app.kubernetes.io/component: webapp
    app.kubernetes.io/part-of: gitops-demo
    app.kubernetes.io/managed-by: argocd
spec:
  replicas: 3
  selector:
    matchLabels:
      app: gitops-demo
  template:
    metadata:
      labels:
        app: gitops-demo
        app.kubernetes.io/name: gitops-demo
        app.kubernetes.io/instance: gitops-demo
        app.kubernetes.io/version: "1.0"
        app.kubernetes.io/component: webapp
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: gitops-demo
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 1001
      containers:
      - name: webapp
        image: ghcr.io/portfolio-jaime/gitops:latest
        imagePullPolicy: Always
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        env:
        - name: NODE_ENV
          value: "production"
        - name: PORT
          value: "8080"
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1001
          capabilities:
            drop:
            - ALL
        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
        startupProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 30
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: var-run
          mountPath: /var/run
        - name: var-cache-nginx
          mountPath: /var/cache/nginx
      volumes:
      - name: tmp
        emptyDir: {}
      - name: var-run
        emptyDir: {}
      - name: var-cache-nginx
        emptyDir: {}
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - gitops-demo
              topologyKey: kubernetes.io/hostname
      terminationGracePeriodSeconds: 30
```

### Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: gitops-demo
  namespace: gitops-demo
  labels:
    app: gitops-demo
    app.kubernetes.io/name: gitops-demo
    app.kubernetes.io/instance: gitops-demo
    app.kubernetes.io/component: webapp
    app.kubernetes.io/part-of: gitops-demo
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: http
    protocol: TCP
    name: http
  selector:
    app: gitops-demo
```

### ServiceAccount y RBAC
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitops-demo
  namespace: gitops-demo
  labels:
    app: gitops-demo
    app.kubernetes.io/name: gitops-demo
    app.kubernetes.io/component: serviceaccount
    app.kubernetes.io/part-of: gitops-demo
automountServiceAccountToken: false
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: gitops-demo
  namespace: gitops-demo
  labels:
    app: gitops-demo
    app.kubernetes.io/name: gitops-demo
    app.kubernetes.io/component: rbac
    app.kubernetes.io/part-of: gitops-demo
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["events"]
  verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: gitops-demo
  namespace: gitops-demo
  labels:
    app: gitops-demo
    app.kubernetes.io/name: gitops-demo
    app.kubernetes.io/component: rbac
    app.kubernetes.io/part-of: gitops-demo
subjects:
- kind: ServiceAccount
  name: gitops-demo
  namespace: gitops-demo
roleRef:
  kind: Role
  name: gitops-demo
  apiGroup: rbac.authorization.k8s.io
```

## Configuraciones Avanzadas

### HorizontalPodAutoscaler
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: gitops-demo-hpa
  namespace: gitops-demo
  labels:
    app: gitops-demo
    app.kubernetes.io/name: gitops-demo
    app.kubernetes.io/component: autoscaler
    app.kubernetes.io/part-of: gitops-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: gitops-demo
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
```

### PodDisruptionBudget
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: gitops-demo-pdb
  namespace: gitops-demo
  labels:
    app: gitops-demo
    app.kubernetes.io/name: gitops-demo
    app.kubernetes.io/component: pdb
    app.kubernetes.io/part-of: gitops-demo
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: gitops-demo
```

### NetworkPolicy
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: gitops-demo-netpol
  namespace: gitops-demo
  labels:
    app: gitops-demo
    app.kubernetes.io/name: gitops-demo
    app.kubernetes.io/component: network-policy
    app.kubernetes.io/part-of: gitops-demo
spec:
  podSelector:
    matchLabels:
      app: gitops-demo
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to: []
    ports:
    - protocol: TCP
      port: 53
    - protocol: UDP
      port: 53
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: TCP
      port: 443
```

## Configuración de Ingress

### Nginx Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gitops-demo
  namespace: gitops-demo
  labels:
    app: gitops-demo
    app.kubernetes.io/name: gitops-demo
    app.kubernetes.io/component: ingress
    app.kubernetes.io/part-of: gitops-demo
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/configuration-snippet: |
      add_header X-Frame-Options "SAMEORIGIN" always;
      add_header X-Content-Type-Options "nosniff" always;
      add_header X-XSS-Protection "1; mode=block" always;
spec:
  tls:
  - hosts:
    - gitops-demo.example.com
    secretName: gitops-demo-tls
  rules:
  - host: gitops-demo.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gitops-demo
            port:
              number: 80
```

### Istio VirtualService
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: gitops-demo
  namespace: gitops-demo
  labels:
    app: gitops-demo
    app.kubernetes.io/name: gitops-demo
    app.kubernetes.io/component: virtualservice
    app.kubernetes.io/part-of: gitops-demo
spec:
  hosts:
  - gitops-demo.example.com
  gateways:
  - gitops-demo-gateway
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: gitops-demo
        port:
          number: 80
    retries:
      attempts: 3
      perTryTimeout: 2s
    timeout: 10s
---
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: gitops-demo-gateway
  namespace: gitops-demo
  labels:
    app: gitops-demo
    app.kubernetes.io/name: gitops-demo
    app.kubernetes.io/component: gateway
    app.kubernetes.io/part-of: gitops-demo
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: gitops-demo-tls
    hosts:
    - gitops-demo.example.com
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - gitops-demo.example.com
    tls:
      httpsRedirect: true
```

## Monitoreo y Observabilidad

### ServiceMonitor (Prometheus)
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: gitops-demo
  namespace: gitops-demo
  labels:
    app: gitops-demo
    app.kubernetes.io/name: gitops-demo
    app.kubernetes.io/component: monitoring
    app.kubernetes.io/part-of: gitops-demo
spec:
  selector:
    matchLabels:
      app: gitops-demo
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
    scrapeTimeout: 10s
```

### ConfigMap para Configuración
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gitops-demo-config
  namespace: gitops-demo
  labels:
    app: gitops-demo
    app.kubernetes.io/name: gitops-demo
    app.kubernetes.io/component: config
    app.kubernetes.io/part-of: gitops-demo
data:
  nginx.conf: |
    events {
        worker_connections 1024;
    }
    http {
        include /etc/nginx/mime.types;
        default_type application/octet-stream;
        
        log_format json_combined escape=json
          '{'
            '"time_local":"$time_local",'
            '"remote_addr":"$remote_addr",'
            '"remote_user":"$remote_user",'
            '"request":"$request",'
            '"status": "$status",'
            '"body_bytes_sent":"$body_bytes_sent",'
            '"request_time":"$request_time",'
            '"http_referrer":"$http_referer",'
            '"http_user_agent":"$http_user_agent"'
          '}';
        
        access_log /var/log/nginx/access.log json_combined;
        error_log /var/log/nginx/error.log warn;
        
        sendfile on;
        keepalive_timeout 65;
        
        server {
            listen 8080;
            server_name localhost;
            root /usr/share/nginx/html;
            index index.html;
            
            location / {
                try_files $uri $uri/ =404;
            }
            
            location /health {
                access_log off;
                return 200 "healthy\n";
                add_header Content-Type text/plain;
            }
            
            location /metrics {
                access_log off;
                return 200 "# HELP nginx_up Nginx status\n# TYPE nginx_up gauge\nnginx_up 1\n";
                add_header Content-Type text/plain;
            }
        }
    }
```

## Comandos Útiles

### Despliegue
```bash
# Aplicar todos los manifiestos
kubectl apply -f Kubernetes/ -n gitops-demo

# Verificar estado del despliegue
kubectl rollout status deployment/gitops-demo -n gitops-demo

# Ver logs
kubectl logs -l app=gitops-demo -n gitops-demo --tail=100 -f

# Describir deployment
kubectl describe deployment gitops-demo -n gitops-demo
```

### Debugging
```bash
# Exec en pod
kubectl exec -it deployment/gitops-demo -n gitops-demo -- /bin/sh

# Port forward para testing local
kubectl port-forward svc/gitops-demo 8080:80 -n gitops-demo

# Ver eventos
kubectl get events -n gitops-demo --sort-by=.metadata.creationTimestamp

# Verificar recursos
kubectl top pods -n gitops-demo
```

### Limpieza
```bash
# Eliminar deployment
kubectl delete deployment gitops-demo -n gitops-demo

# Eliminar todo el namespace
kubectl delete namespace gitops-demo

# Eliminar recursos específicos
kubectl delete -f Kubernetes/ -n gitops-demo
```