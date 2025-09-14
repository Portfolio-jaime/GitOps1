# GitHub Actions CI/CD Pipeline

## Configuración Principal

### Workflow CI/CD (.github/workflows/cd.yml)
```yaml
name: GitOps CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        default: 'development'
        type: choice
        options:
        - development
        - staging
        - production

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
  DOCKER_BUILDKIT: 1

jobs:
  # Job de análisis de código y seguridad
  security-scan:
    name: Security Analysis
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      with:
        fetch-depth: 0

    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        scan-ref: '.'
        format: 'sarif'
        output: 'trivy-results.sarif'

    - name: Upload Trivy scan results
      uses: github/codeql-action/upload-sarif@v3
      if: always()
      with:
        sarif_file: 'trivy-results.sarif'

    - name: Run Hadolint
      uses: hadolint/hadolint-action@v3.1.0
      with:
        dockerfile: Docker/Dockerfile
        format: sarif
        output-file: hadolint-results.sarif
        no-fail: true

    - name: Upload Hadolint results
      uses: github/codeql-action/upload-sarif@v3
      if: always()
      with:
        sarif_file: hadolint-results.sarif

  # Job de testing
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    needs: [security-scan]
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '18'
        cache: 'npm'

    - name: Install dependencies
      run: npm ci

    - name: Run unit tests
      run: npm test

    - name: Run integration tests
      run: npm run test:integration

    - name: Upload test results
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: test-results
        path: |
          coverage/
          test-results.xml

  # Job de construcción y push de imagen
  build-and-push:
    name: Build and Push Docker Image
    runs-on: ubuntu-latest
    needs: [test]
    permissions:
      contents: read
      packages: write
    
    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
      image-tags: ${{ steps.meta.outputs.tags }}

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Log in to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=sha,prefix={{branch}}-
          type=raw,value=latest,enable={{is_default_branch}}
          type=raw,value={{date 'YYYY-MM-DD-HHmmss'}}
        labels: |
          org.opencontainers.image.title=GitOps Demo Application
          org.opencontainers.image.description=Simple web application for GitOps demonstration
          org.opencontainers.image.vendor=Portfolio-jaime
          org.opencontainers.image.licenses=MIT

    - name: Build and push Docker image
      id: build
      uses: docker/build-push-action@v5
      with:
        context: .
        file: Docker/Dockerfile
        platforms: linux/amd64,linux/arm64
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
        provenance: false
        sbom: true

    - name: Run Trivy scanner on image
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
        format: 'sarif'
        output: 'trivy-image-results.sarif'

    - name: Upload Trivy scan results
      uses: github/codeql-action/upload-sarif@v3
      if: always()
      with:
        sarif_file: 'trivy-image-results.sarif'

  # Job de actualización de manifiestos
  update-manifests:
    name: Update Kubernetes Manifests
    runs-on: ubuntu-latest
    needs: [build-and-push]
    if: github.ref == 'refs/heads/main' || github.event_name == 'workflow_dispatch'
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      with:
        token: ${{ secrets.GITHUB_TOKEN }}
        fetch-depth: 0

    - name: Configure Git
      run: |
        git config --local user.email "action@github.com"
        git config --local user.name "GitHub Action"

    - name: Update deployment manifest
      run: |
        IMAGE_TAG="${{ github.sha }}"
        if [[ "${{ github.event_name }}" == "workflow_dispatch" ]]; then
          IMAGE_TAG="${{ github.event.inputs.environment }}-${{ github.sha }}"
        fi
        
        sed -i "s|image: .*|image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${IMAGE_TAG}|" Kubernetes/deployment.yaml
        
        echo "Updated image tag to: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${IMAGE_TAG}"

    - name: Commit and push changes
      run: |
        git add Kubernetes/deployment.yaml
        if git diff --staged --quiet; then
          echo "No changes to commit"
        else
          git commit -m "🤖 Update image tag to ${{ github.sha }}
          
          - Updated deployment manifest with new image
          - Triggered by: ${{ github.event_name }}
          - Commit: ${{ github.sha }}
          - Workflow: ${{ github.workflow }}"
          
          git push origin main
        fi

  # Job de notificaciones
  notify:
    name: Send Notifications
    runs-on: ubuntu-latest
    needs: [build-and-push, update-manifests]
    if: always()
    
    steps:
    - name: Notify Slack on Success
      if: needs.build-and-push.result == 'success' && needs.update-manifests.result == 'success'
      uses: rtCamp/action-slack-notify@v2
      env:
        SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
        SLACK_CHANNEL: gitops-deployments
        SLACK_COLOR: good
        SLACK_MESSAGE: |
          ✅ GitOps deployment successful!
          
          Repository: ${{ github.repository }}
          Branch: ${{ github.ref_name }}
          Commit: ${{ github.sha }}
          Image: ${{ needs.build-and-push.outputs.image-tags }}
          
          ArgoCD will automatically sync the changes.

    - name: Notify Slack on Failure
      if: needs.build-and-push.result == 'failure' || needs.update-manifests.result == 'failure'
      uses: rtCamp/action-slack-notify@v2
      env:
        SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
        SLACK_CHANNEL: gitops-deployments
        SLACK_COLOR: danger
        SLACK_MESSAGE: |
          ❌ GitOps deployment failed!
          
          Repository: ${{ github.repository }}
          Branch: ${{ github.ref_name }}
          Commit: ${{ github.sha }}
          
          Please check the workflow logs for details.
```

## Workflows Adicionales

### Workflow de Release (.github/workflows/release.yml)
```yaml
name: Create Release

on:
  push:
    tags:
      - 'v*.*.*'

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  create-release:
    name: Create GitHub Release
    runs-on: ubuntu-latest
    permissions:
      contents: write
      packages: write
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      with:
        fetch-depth: 0

    - name: Generate changelog
      id: changelog
      run: |
        # Generar changelog basado en commits
        PREVIOUS_TAG=$(git describe --tags --abbrev=0 HEAD^ 2>/dev/null || echo "")
        if [ -z "$PREVIOUS_TAG" ]; then
          CHANGELOG=$(git log --pretty=format:"- %s" --no-merges)
        else
          CHANGELOG=$(git log ${PREVIOUS_TAG}..HEAD --pretty=format:"- %s" --no-merges)
        fi
        
        echo "CHANGELOG<<EOF" >> $GITHUB_OUTPUT
        echo "$CHANGELOG" >> $GITHUB_OUTPUT
        echo "EOF" >> $GITHUB_OUTPUT

    - name: Create Release
      uses: softprops/action-gh-release@v1
      with:
        name: Release ${{ github.ref_name }}
        body: |
          ## Changes in this release
          
          ${{ steps.changelog.outputs.CHANGELOG }}
          
          ## Docker Images
          
          - `${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.ref_name }}`
          - `${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest`
          
          ## Deployment
          
          This release will be automatically deployed via ArgoCD.
        draft: false
        prerelease: false
        generate_release_notes: true

  build-release-image:
    name: Build Release Image
    runs-on: ubuntu-latest
    needs: [create-release]
    permissions:
      contents: read
      packages: write
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Log in to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Extract version from tag
      id: version
      run: echo "VERSION=${GITHUB_REF#refs/tags/}" >> $GITHUB_OUTPUT

    - name: Build and push release image
      uses: docker/build-push-action@v5
      with:
        context: .
        file: Docker/Dockerfile
        platforms: linux/amd64,linux/arm64
        push: true
        tags: |
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.version.outputs.VERSION }}
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
        labels: |
          org.opencontainers.image.title=GitOps Demo Application
          org.opencontainers.image.description=Simple web application for GitOps demonstration
          org.opencontainers.image.version=${{ steps.version.outputs.VERSION }}
          org.opencontainers.image.vendor=Portfolio-jaime
        cache-from: type=gha
        cache-to: type=gha,mode=max
```

### Workflow de Cleanup (.github/workflows/cleanup.yml)
```yaml
name: Cleanup Old Images

on:
  schedule:
    - cron: '0 2 * * 0'  # Weekly on Sunday at 2 AM
  workflow_dispatch:

jobs:
  cleanup-images:
    name: Delete Old Container Images
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    
    steps:
    - name: Delete old container images
      uses: actions/delete-package-versions@v4
      with:
        package-name: ${{ github.event.repository.name }}
        package-type: 'container'
        min-versions-to-keep: 10
        delete-only-untagged-versions: false
        ignore-versions: '^latest$|^v\d+\.\d+\.\d+$'
```

## Configuraciones de Seguridad

### Dependabot (.github/dependabot.yml)
```yaml
version: 2
updates:
  - package-ecosystem: "docker"
    directory: "/Docker"
    schedule:
      interval: "weekly"
    reviewers:
      - "Portfolio-jaime"
    labels:
      - "dependencies"
      - "docker"

  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    reviewers:
      - "Portfolio-jaime"
    labels:
      - "dependencies"
      - "javascript"

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    reviewers:
      - "Portfolio-jaime"
    labels:
      - "dependencies"
      - "github-actions"
```

### Branch Protection Rules
```json
{
  "required_status_checks": {
    "strict": true,
    "contexts": [
      "Security Analysis",
      "Run Tests",
      "Build and Push Docker Image"
    ]
  },
  "enforce_admins": false,
  "required_pull_request_reviews": {
    "required_approving_review_count": 1,
    "dismiss_stale_reviews": true,
    "require_code_owner_reviews": true
  },
  "restrictions": null,
  "allow_force_pushes": false,
  "allow_deletions": false,
  "block_creations": false,
  "required_conversation_resolution": true
}
```

## Secrets y Variables

### Repository Secrets
```bash
# Secrets requeridos en GitHub
SLACK_WEBHOOK=https://hooks.slack.com/services/...
ARGOCD_SERVER=argocd.example.com
ARGOCD_AUTH_TOKEN=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# Variables de entorno
REGISTRY_URL=ghcr.io
DEPLOYMENT_ENVIRONMENT=production
MONITORING_ENABLED=true
```

### Environment-specific Secrets
```yaml
# Para diferentes entornos
environments:
  development:
    secrets:
      KUBE_CONFIG: ${{ secrets.DEV_KUBE_CONFIG }}
      REGISTRY: ghcr.io
  
  staging:
    secrets:
      KUBE_CONFIG: ${{ secrets.STAGING_KUBE_CONFIG }}
      REGISTRY: ghcr.io
  
  production:
    secrets:
      KUBE_CONFIG: ${{ secrets.PROD_KUBE_CONFIG }}
      REGISTRY: ghcr.io
    protection_rules:
      required_reviewers: 2
```

## Templates de Issues y PRs

### Pull Request Template (.github/pull_request_template.md)
```markdown
## Description
Brief description of the changes in this PR.

## Type of Change
- [ ] Bug fix (non-breaking change which fixes an issue)
- [ ] New feature (non-breaking change which adds functionality)  
- [ ] Breaking change (fix or feature that would cause existing functionality to not work as expected)
- [ ] Documentation update
- [ ] Infrastructure change

## Testing
- [ ] Unit tests pass locally
- [ ] Integration tests pass locally
- [ ] Manual testing completed
- [ ] Security scan completed

## Deployment
- [ ] Changes are backward compatible
- [ ] Database migrations included (if applicable)
- [ ] Configuration changes documented
- [ ] Monitoring/alerting updated (if applicable)

## Checklist
- [ ] Code follows the project's style guidelines
- [ ] Self-review of code completed
- [ ] Code is properly commented
- [ ] Documentation updated
- [ ] No new warnings introduced
```

### Issue Templates (.github/ISSUE_TEMPLATE/)

#### Bug Report (bug_report.yml)
```yaml
name: Bug Report
description: Create a bug report to help improve the project
title: "[BUG] "
labels: ["bug", "triage"]
body:
  - type: markdown
    attributes:
      value: |
        Thank you for reporting a bug! Please fill out the information below.

  - type: textarea
    id: description
    attributes:
      label: Description
      description: Clear description of the bug
    validations:
      required: true

  - type: textarea
    id: reproduction
    attributes:
      label: Steps to Reproduce
      description: Steps to reproduce the behavior
    validations:
      required: true

  - type: textarea
    id: expected
    attributes:
      label: Expected Behavior
      description: What you expected to happen
    validations:
      required: true

  - type: textarea
    id: environment
    attributes:
      label: Environment
      description: |
        - Kubernetes version:
        - ArgoCD version:
        - Browser:
      value: |
        - Kubernetes version:
        - ArgoCD version:
        - Browser:
    validations:
      required: true
```

## Monitoreo de Workflows

### Métricas de CI/CD
- **Build Duration**: Tiempo de construcción promedio
- **Success Rate**: Tasa de éxito de deployments
- **Lead Time**: Tiempo desde commit hasta deployment
- **Frequency**: Frecuencia de deployments

### Alertas
- Fallos en pipeline
- Vulnerabilidades detectadas
- Tiempo de build excesivo
- Fallos en tests