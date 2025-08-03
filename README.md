# CI/CD Tools

A reusable GitHub Actions CI/CD pipeline for containerized applications using GitOps with ArgoCD for Azure Kubernetes Service (AKS).

## Overview

This repository provides reusable GitHub Actions workflows that implement a GitOps CI/CD pipeline:

**Build → Update GitOps Manifests → ArgoCD Auto-Deploy**

The pipeline is designed to:
- Build Docker images and push them to Azure Container Registry (ACR)
- Update GitOps manifests in a separate repository
- Let ArgoCD automatically deploy applications to Azure Kubernetes Service (AKS)
- Support multiple environments (dev, production)
- Follow GitOps best practices with declarative deployments

## Supported Branches

| Branch | Environment | Image Tag | Namespace |
|--------|-------------|-----------|-----------|
| `dev` | Development | `vars.IMAGE_TAG_NP` | `vars.NAMESPACE_NAME_NP` |
| `main` | Production | `vars.IMAGE_TAG_PROD` | `vars.NAMESPACE_NAME_PROD` |

## Quick Start

### 1. Add to Your Application Repository

In your application repository, create `.github/workflows/main.yml`:

```yaml
name: CI/CD Pipeline
on:
  push:
    branches: ["dev"]
  pull_request:
    branches: ["main", "dev"]

permissions:
  contents: read
  id-token: write
  actions: write 

jobs:
  call-full-pipeline:
    name: Run Full CI/CD Pipeline
    uses: Nexus-Team-Project/cicd-tools/.github/workflows/main.yml@main
    secrets: inherit
```

**Note**: The workflow automatically uses your repository variables. No inputs needed!

### 2. Required Repository Variables

Add these repository/organization variables:

- `AZURE_CONTAINER_REGISTRY` - Name of your ACR
- `RESOURCE_GROUP` - Resource group containing your ACR
- `IMAGE_TAG_NP` - Image tag for non-production (e.g., "dev")
- `IMAGE_TAG_PROD` - Image tag for production (e.g., "stable")
- `NAMESPACE_NAME_NP` - Kubernetes namespace for non-production (e.g., "dev")
- `NAMESPACE_NAME_PROD` - Kubernetes namespace for production (e.g., "production")

### 3. Required Secrets

Add these secrets to your repository:

- `AZURE_CLIENT_ID` - Azure service principal client ID
- `AZURE_TENANT_ID` - Azure tenant ID  
- `AZURE_SUBSCRIPTION_ID` - Azure subscription ID
- `ARGOCD_REPO_TOKEN` - GitHub token with write access to ArgoCD repository (optional, uses GITHUB_TOKEN if not provided)

### 4. Required Files

Your repository needs:

- `Dockerfile` - Container definition
- Application code that runs on configured port
- Health check endpoints (recommended):
  - `/` - For liveness probe
  - `/` - For readiness probe

### 5. ArgoCD Application Setup

For **new services**, create an ArgoCD Application in your cluster (one-time setup):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/Nexus-Team-Project/argocd-apps
    targetRevision: HEAD
    path: manifests/dev/my-app/k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Apply once: `kubectl apply -f application.yaml`

## How It Works (GitOps Flow)

### 1. Rules Evaluation (`.github/workflows/rules.yml`)
- Determines if CI/CD should run based on branch and event type
- Only runs on `push` events to supported branches
- Sets target environment, image tag, and namespace

### 2. Build Orchestration (`.github/workflows/build-orchestrator.yml`)
- Builds Docker image using Azure Container Registry build
- Auto-detects ACR if not specified
- Tags images appropriately for the environment

### 3. GitOps Update (`.github/workflows/update-argocd.yml`)
- Updates image tag in GitOps manifests repository
- Checks out ArgoCD apps repository
- Updates `deployment.yaml` with new image tag
- Commits and pushes changes to GitOps repo
- **No direct Kubernetes deployment**

### 4. ArgoCD Auto-Deployment
- ArgoCD detects Git changes automatically
- Syncs Kubernetes cluster with updated manifests
- Handles rollouts, health checks, and status reporting
- Monitor at: https://dev.nexus-online.net/argocd/

## GitOps Repository Structure

The ArgoCD manifests repository should follow this structure:

```
manifests/
├── dev/
│   └── my-app/
│       ├── application.yaml    # ArgoCD Application definition
│       └── k8s/               # Kubernetes manifests
│           ├── deployment.yaml
│           └── service.yaml
└── prod/
    └── my-app/
        ├── application.yaml
        └── k8s/
            ├── deployment.yaml
            └── service.yaml
```

## Kubernetes Resources

ArgoCD will deploy these Kubernetes resources (configure in GitOps repo):

### Deployment
- Configurable replicas
- Health checks on your endpoints
- Resource limits as configured
- Environment variables as needed

### Service
- ClusterIP service exposing your application port
- Named port for better networking

## Advanced Configuration

### Custom ArgoCD Repository

You can customize the GitOps repository:

```yaml
jobs:
  call-full-pipeline:
    uses: Nexus-Team-Project/cicd-tools/.github/workflows/main.yml@main
    with:
      argocd_repo: 'MyOrg/my-gitops-repo'
      argocd_repo_branch: 'main'
    secrets: inherit
```

### Environment-Specific Behavior

The pipeline automatically:
- Uses dev environment for `dev` branch pushes
- Uses production environment for `main` branch pushes
- Only deploys on supported branches
- Skips GitOps update if build fails

## Azure Prerequisites

1. **Azure Container Registry (ACR)**
   - Must be accessible by the service principal
   - Can be auto-detected or specified explicitly

2. **Azure Kubernetes Service (AKS)**
   - Should have ArgoCD installed and configured
   - ArgoCD should have access to your GitOps repository

3. **Service Principal**
   - Needs `AcrPush` role on ACR
   - Needs `Reader` role on resource groups (for auto-detection)

## ArgoCD Prerequisites

1. **ArgoCD Installation**
   - ArgoCD installed in your cluster
   - Access to ArgoCD UI for monitoring

2. **GitOps Repository Access**
   - ArgoCD configured with access to your manifests repository
   - Repository credentials configured in ArgoCD

3. **Application Setup**
   - ArgoCD Application resources created for each app/environment
   - Proper App of Apps pattern implementation (recommended)

## Troubleshooting

### Build Issues
- Ensure Dockerfile exists in repository root
- Check ACR permissions for service principal
- Verify Azure credentials are correct

### GitOps Update Issues
- Check GitHub token permissions for ArgoCD repository
- Verify ArgoCD repository structure matches expected paths
- Ensure manifest files exist in GitOps repository

### ArgoCD Deployment Issues
- Check ArgoCD Application status in UI
- Verify ArgoCD has access to GitOps repository
- Check Kubernetes RBAC permissions for ArgoCD
- Monitor ArgoCD logs for sync issues

### Branch Not Triggering
- Only `main` and `dev` branches trigger CI/CD
- Only `push` events trigger the pipeline
- Check branch name spelling

## Benefits of GitOps Approach

- **Declarative**: Infrastructure as Code with Git as source of truth
- **Auditable**: All changes tracked in Git history
- **Rollback**: Easy rollback using Git operations
- **Security**: No cluster credentials in CI/CD pipelines
- **Consistency**: Same deployment process across all environments
- **Observability**: ArgoCD provides deployment status and health monitoring

## Contributing

To modify the CI/CD pipeline:

1. Fork this repository
2. Make changes to workflow files
3. Test with a feature branch and sample application.
4. Submit pull request

## Repository Information

- **ArgoCD Repository**: https://github.com/Nexus-Team-Project/argocd-apps
- **ArgoCD UI**: https://dev.nexus-online.net/argocd/