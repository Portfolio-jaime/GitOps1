# Proyecto GitOps1 con Helm, Ingress y GitHub Actions

Este proyecto ahora utiliza Helm para la gestión de despliegues en Kubernetes, con soporte para Ingress y pipelines modernos en GitHub Actions.

## Estructura del Proyecto

```
GitOps1/
├── chart/                # Helm chart principal (Deployment, Service, Ingress)
├── Docker/               # Dockerfile y recursos de la app
├── legacy-manifests/     # Manifiestos antiguos (solo referencia histórica)
├── .github/workflows/    # Pipeline CD.yml actualizado para Helm
├── README.md             # Esta documentación
└── ...
```

## Despliegue con Helm

1. Construye y sube la imagen Docker (esto lo hace el pipeline automáticamente):

```sh
# (Opcional) Build y push manual
cd Docker
DOCKERHUB_USER=usuario
IMAGE_NAME=gitops1
TAG=latest

docker build -t $DOCKERHUB_USER/$IMAGE_NAME:$TAG .
docker push $DOCKERHUB_USER/$IMAGE_NAME:$TAG
```

2. Despliega usando Helm:

```sh
cd GitOps1
helm install gitops1 ./chart \
  --set image.repository=usuario/gitops1 \
  --set image.tag=latest
```

3. (Opcional) Personaliza el Ingress en `values.yaml`:

```yaml
ingress:
  enabled: true
  hosts:
    - host: tu-dominio.local
      paths:
        - path: /
          pathType: ImplementationSpecific
```

## Pipeline CI/CD

El workflow `.github/workflows/CD.yml` ahora:
- Construye y sube la imagen Docker.
- Lint y template del chart Helm.
- (Opcional) Despliegue automático con Helm si se configura el kubeconfig.

## Migración y manifiestos antiguos

Los manifiestos YAML directos (`deployment.yaml`, `service.yaml`) han sido movidos a `legacy-manifests/` y ya no se usan para despliegue. Todo el despliegue se gestiona vía Helm.

## Documentación adicional

- El chart Helm soporta configuración de réplicas, imagen, puertos y reglas de Ingress.
- Para usar ArgoCD, apunta la ruta de la app a `chart/` en vez de `Kubernetes/`.

---

Jaime A. Henao
Cloud Engineer.
