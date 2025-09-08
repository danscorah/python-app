# ArgoCD + GitHub Actions CI/CD Setup Guide

## Overview

This guide shows how to set up a complete GitOps CI/CD pipeline using GitHub Actions and ArgoCD.

## Architecture

```
Developer Push → GitHub Actions → Git Repository → ArgoCD → Kubernetes
     ↓                ↓              ↓              ↓
  Code Change    Build & Test   Update Manifests   Deploy App
```

## Step 1: Prerequisites

### 1.1 Kubernetes Cluster
```bash
# Using Kind (for local development)
kind create cluster --name gitops-demo

# Or use any Kubernetes cluster (GKE, EKS, AKS, etc.)
```

### 1.2 Install ArgoCD
```bash
# Create ArgoCD namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

### 1.3 Expose ArgoCD UI
```bash
# Port forward to access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Access UI at https://localhost:8080
# Username: admin
# Password: (from step 1.2)
```

## Step 2: GitHub Repository Setup

### 2.1 Repository Structure
```
my-app/
├── .github/workflows/ci-cd.yaml
├── src/
│   └── app.py
├── charts/
│   └── my-app/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           ├── service.yaml
│           └── ingress.yaml
└── Dockerfile
```

### 2.2 GitHub Secrets
Add these secrets to your repository (Settings → Secrets and variables → Actions):

- `DOCKERHUB_USERNAME`: Your Docker Hub username
- `DOCKERHUB_TOKEN`: Your Docker Hub access token
- `GITHUB_TOKEN`: (Automatically provided by GitHub)

## Step 3: GitHub Actions Workflow

### 3.1 CI/CD Pipeline
```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
    paths: ['src/**']

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/my-app:${{ github.sha }}
          
  cd:
    needs: ci
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Update Helm values
        run: |
          # Update image tag in values.yaml
          sed -i "s|imageTag:.*|imageTag: ${{ github.sha }}|g" charts/my-app/values.yaml
          
      - name: Commit and push changes
        uses: EndBug/add-and-commit@v9
        with:
          message: "Update image to ${{ github.sha }}"
```

## Step 4: ArgoCD Application

### 4.1 Create Application
```bash
# Create ArgoCD application
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-username/my-app
    targetRevision: HEAD
    path: charts/my-app
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF
```

### 4.2 Alternative: Create via UI
1. Go to ArgoCD UI (https://localhost:8080)
2. Click "New App"
3. Fill in:
   - Application Name: `my-app`
   - Project: `default`
   - Repository URL: `https://github.com/your-username/my-app`
   - Path: `charts/my-app`
   - Cluster URL: `https://kubernetes.default.svc`
   - Namespace: `default`
4. Enable "Auto Sync" and "Self Heal"

## Step 5: How It Works

### 5.1 Complete Flow
1. **Developer pushes code** to `src/` directory
2. **GitHub Actions triggers** CI job
3. **CI builds Docker image** and pushes to registry
4. **CD job updates** Helm values with new image tag
5. **CD commits and pushes** changes to Git
6. **ArgoCD detects** Git changes automatically
7. **ArgoCD deploys** new version to Kubernetes

### 5.2 Key Benefits
- **Git as Source of Truth**: All changes tracked in Git
- **Automatic Deployment**: No manual intervention needed
- **Rollback Capability**: Easy to revert to previous versions
- **Audit Trail**: Complete history of all changes
- **Multi-Environment**: Easy to promote across environments

## Step 6: Advanced Features

### 6.1 Multi-Environment Setup
```yaml
# values-dev.yaml
imageTag: latest
replicas: 1

# values-staging.yaml  
imageTag: latest
replicas: 2

# values-prod.yaml
imageTag: v1.2.3
replicas: 3
```

### 6.2 ArgoCD ApplicationSets
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          environment: dev
  template:
    metadata:
      name: '{{name}}-my-app'
    spec:
      source:
        repoURL: https://github.com/your-username/my-app
        path: charts/my-app
        targetRevision: HEAD
      destination:
        server: '{{server}}'
        namespace: default
```

## Step 7: Monitoring and Troubleshooting

### 7.1 ArgoCD CLI
```bash
# Install ArgoCD CLI
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/

# Login
argocd login localhost:8080

# Check application status
argocd app get my-app

# Sync application
argocd app sync my-app
```

### 7.2 Common Issues
- **Sync Issues**: Check ArgoCD logs and application events
- **Permission Issues**: Ensure GitHub token has write permissions
- **Image Pull Issues**: Check image registry credentials
- **Helm Issues**: Validate Helm chart syntax

## Step 8: Best Practices

1. **Use Semantic Versioning** for image tags
2. **Implement Proper Testing** in CI pipeline
3. **Use Secrets Management** for sensitive data
4. **Monitor Application Health** with proper probes
5. **Implement Blue-Green or Canary** deployments for production
6. **Use ArgoCD Projects** for multi-tenancy
7. **Implement RBAC** for ArgoCD access control

This setup provides a robust, scalable GitOps CI/CD pipeline that's commonly used in production environments.
