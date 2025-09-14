# Docker Configuration

## Dockerfile

### Configuración de la Imagen
```dockerfile
FROM nginx:alpine

# Metadatos de la imagen
LABEL maintainer="jaime.andres.henao.arbelaez@ba.com"
LABEL description="GitOps Demo - Simple web application"
LABEL version="1.0"

# Copiar aplicación web
COPY Docker/index.html /usr/share/nginx/html/

# Configuración de Nginx
COPY Docker/nginx.conf /etc/nginx/nginx.conf

# Exponer puerto
EXPOSE 80

# Usuario no-root para seguridad
RUN addgroup -g 1001 -S nginx && \
    adduser -S -D -H -u 1001 -h /var/cache/nginx -s /sbin/nologin -G nginx -g nginx nginx

# Cambiar ownership de archivos
RUN chown -R nginx:nginx /usr/share/nginx/html && \
    chown -R nginx:nginx /var/cache/nginx && \
    chown -R nginx:nginx /var/log/nginx && \
    chown -R nginx:nginx /etc/nginx/conf.d

# Cambiar a usuario no-root
USER nginx

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost/ || exit 1

# Comando por defecto
CMD ["nginx", "-g", "daemon off;"]
```

## Configuración de Nginx

### nginx.conf personalizado
```nginx
events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    # Configuración de logging
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;

    # Configuración de rendimiento
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    # Configuración de compresión
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/javascript
        application/xml+rss
        application/json;

    server {
        listen 8080;
        server_name localhost;
        root /usr/share/nginx/html;
        index index.html;

        # Configuración de seguridad
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header Referrer-Policy "no-referrer-when-downgrade" always;
        add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;

        # Configuración de caché
        location ~* \.(css|js|png|jpg|jpeg|gif|ico|svg)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }

        # Ruta principal
        location / {
            try_files $uri $uri/ =404;
        }

        # Health check endpoint
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }

        # Configuración de error pages
        error_page 404 /404.html;
        error_page 500 502 503 504 /50x.html;
        
        location = /50x.html {
            root /usr/share/nginx/html;
        }
    }
}
```

## Build Strategy

### Multi-stage Build (Optimizado)
```dockerfile
# Build stage
FROM node:16-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Production stage
FROM nginx:alpine

# Instalar curl para health checks
RUN apk add --no-cache curl

# Copiar archivos de aplicación
COPY --from=builder /app/dist /usr/share/nginx/html
COPY Docker/nginx.conf /etc/nginx/nginx.conf

# Configuración de seguridad
RUN addgroup -g 1001 -S nginx && \
    adduser -S -D -H -u 1001 -h /var/cache/nginx -s /sbin/nologin -G nginx -g nginx nginx && \
    chown -R nginx:nginx /usr/share/nginx/html && \
    chown -R nginx:nginx /var/cache/nginx && \
    chown -R nginx:nginx /var/log/nginx && \
    chown -R nginx:nginx /etc/nginx/conf.d && \
    touch /var/run/nginx.pid && \
    chown -R nginx:nginx /var/run/nginx.pid

USER nginx
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

CMD ["nginx", "-g", "daemon off;"]
```

## Docker Compose para Desarrollo

### docker-compose.yml
```yaml
version: '3.8'

services:
  webapp:
    build:
      context: .
      dockerfile: Docker/Dockerfile
    container_name: gitops-demo
    ports:
      - "8080:8080"
    environment:
      - NODE_ENV=development
    volumes:
      - ./Docker/index.html:/usr/share/nginx/html/index.html:ro
    networks:
      - gitops-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  # Opcional: Agregar un reverse proxy
  traefik:
    image: traefik:v2.10
    command:
      - --api.insecure=true
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --entrypoints.web.address=:80
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - gitops-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.traefik.rule=Host(`traefik.localhost`)"
      - "traefik.http.routers.traefik.entrypoints=web"

networks:
  gitops-network:
    driver: bridge
```

## Optimizaciones de Imagen

### .dockerignore
```gitignore
# Git
.git
.gitignore

# Documentation
README.md
docs/
*.md

# Development files
docker-compose*.yml
.env*
.vscode/
.idea/

# CI/CD
.github/
Jenkinsfile
.gitlab-ci.yml

# Kubernetes
Kubernetes/
k8s/

# Node modules (if applicable)
node_modules/
npm-debug.log*

# Logs
*.log

# OS generated files
.DS_Store
Thumbs.db

# Temporary files
tmp/
temp/
*.tmp
*.bak
```

### Análisis de Imagen
```bash
# Analizar tamaño de imagen
docker images gitops-demo

# Inspeccionar layers
docker history gitops-demo

# Scan de vulnerabilidades
docker scan gitops-demo

# Analizar con dive
dive gitops-demo
```

## Mejores Prácticas Implementadas

### 1. Seguridad
- **Non-root user**: Usuario no-root para ejecución
- **Security headers**: Headers de seguridad HTTP
- **Health checks**: Verificación de estado
- **Minimal base**: Imagen Alpine para reducir superficie de ataque

### 2. Rendimiento
- **Gzip compression**: Compresión de contenido
- **Static file caching**: Caché de archivos estáticos
- **Keepalive connections**: Conexiones persistentes
- **Multi-stage builds**: Optimización de tamaño

### 3. Observabilidad
- **Structured logging**: Logs estructurados
- **Health endpoint**: Endpoint de salud
- **Metrics ready**: Preparado para métricas
- **Graceful shutdown**: Cierre ordenado

### 4. Portabilidad
- **Environment variables**: Configuración vía variables
- **Volume mounts**: Configuración externa
- **Signal handling**: Manejo de señales del sistema
- **Standard ports**: Puertos estándar

## Scripts de Utilidad

### build.sh
```bash
#!/bin/bash
set -e

# Variables
IMAGE_NAME="gitops-demo"
TAG="${1:-latest}"
REGISTRY="${DOCKER_REGISTRY:-ghcr.io/portfolio-jaime}"

echo "Building image: ${REGISTRY}/${IMAGE_NAME}:${TAG}"

# Build
docker build \
  -t ${IMAGE_NAME}:${TAG} \
  -t ${REGISTRY}/${IMAGE_NAME}:${TAG} \
  -f Docker/Dockerfile \
  .

# Opcional: Push si se proporciona registry
if [ ! -z "$DOCKER_REGISTRY" ]; then
  echo "Pushing to registry..."
  docker push ${REGISTRY}/${IMAGE_NAME}:${TAG}
fi

echo "Build complete!"
```

### run-local.sh
```bash
#!/bin/bash

# Ejecutar localmente para testing
docker run -d \
  --name gitops-demo-local \
  -p 8080:8080 \
  --rm \
  gitops-demo:latest

echo "Application running at: http://localhost:8080"
echo "Health check: http://localhost:8080/health"
echo ""
echo "To stop: docker stop gitops-demo-local"
```