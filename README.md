# CI/CD Tools

A reusable GitHub Actions CI/CD pipeline for containerized applications deployed to Azure Kubernetes Service (AKS).

## Overview

This repository provides reusable GitHub Actions workflows that implement a simple but effective CI/CD pipeline:

**Build → Deploy**

The pipeline is designed to:
- Build Docker images and push them to Azure Container Registry (ACR)
- Deploy applications to Azure Kubernetes Service (AKS) 
- Support multiple environments (dev, production)
- Auto-detect Azure resources when possible

## Supported Branches

| Branch | Environment | Image Tag | Namespace |
|--------|-------------|-----------|-----------|
| `dev` | Development | `vars.IMAGE_TAG_NP` | `vars.NAMESPACE_NAME_NP` |
| `main` | Production | `vars.IMAGE_TAG_PROD` | `vars.NAMESPACE_NAME_PROD` |

## Quick Start

### 1. Add to Your Repository

In your application repository, create `.github/workflows/main.yml`:

```yaml
name: CI/CD Pipeline
on:
  push:
    branches: ["feat/add-ci", "develop"]
  pull_request:
    branches: ["feat/add-ci"]

permissions:
  contents: read
  id-token: write
  actions: write 

  
jobs:
  call-full-pipeline:
    name: Run Full CI/CD Pipeline
    uses: Nexus-Team-Project/cicd-tools/.github/workflows/main.yml@build-basic-cicd
    secrets: inherit
```

**Note**: The workflow automatically uses your repository variables. No inputs needed!

### 3. Required Secrets

Add these secrets to your repository:

Add these repository/organization variables:

- `AZURE_CONTAINER_REGISTRY` - Name of your ACR
- `RESOURCE_GROUP` - Resource group containing your ACR
- `IMAGE_TAG_NP` - Image tag for non-production (e.g., "dev")
- `IMAGE_TAG_PROD` - Image tag for production (e.g., "stable")
- `NAMESPACE_NAME_NP` - Kubernetes namespace for non-production (e.g., "dev")
- `NAMESPACE_NAME_PROD` - Kubernetes namespace for production (e.g., "production")

- `AZURE_CLIENT_ID` - Azure service principal client ID
- `AZURE_TENANT_ID` - Azure tenant ID  
- `AZURE_SUBSCRIPTION_ID` - Azure subscription ID

### 4. Required Files

Your repository needs:

- `Dockerfile` - Container definition
- Application code that runs on port 3000
- Health check endpoints (recommended):
  - `/` - For liveness probe
  - `/` - For readiness probe

## How It Works

### 1. Rules Evaluation (`.github/workflows/rules.yml`)
- Determines if CI/CD should run based on branch and event type
- Only runs on `push` events to supported branches
- Sets target environment, image tag, and namespace

### 2. Build Orchestration (`.github/workflows/build-orchestrator.yml`)
- Builds Docker image using Azure Container Registry build
- Auto-detects ACR if not specified
- Tags images appropriately for the environment

### 3. Deployment (`.github/workflows/deploy.aks.yml`)
- Deploys to AKS using Kubernetes manifests
- Creates namespace if it doesn't exist
- Applies deployment and service configurations
- Waits for deployment to be ready

## Kubernetes Resources

The pipeline creates these Kubernetes resources:

### Deployment
- 2 replicas by default
- Health checks on `/health` and `/ready` endpoints
- Resource limits: 512Mi memory, 500m CPU
- Environment variables: `NODE_ENV`, `PORT`

### Service
- ClusterIP service exposing port 3000
- Named port for better networking

## Advanced Configuration

### Environment-Specific Behavior

The pipeline automatically:
- Uses `dev` tag and namespace for `dev` and `dev` branches
- Uses `stable` tag and `production` namespace for `main` branch
- Only deploys on supported branches
- Skips deployment if build fails

## Azure Prerequisites

1. **Azure Container Registry (ACR)**
   - Must be accessible by the service principal
   - Can be auto-detected or specified explicitly

2. **Azure Kubernetes Service (AKS)**
   - Must be accessible by the service principal
   - Should have appropriate RBAC permissions

3. **Service Principal**
   - Needs `AcrPush` role on ACR
   - Needs `Azure Kubernetes Service Cluster User Role` on AKS
   - Needs `Reader` role on resource groups (for auto-detection)

## Troubleshooting

### Build Issues
- Ensure Dockerfile exists in repository root
- Check ACR permissions for service principal
- Verify Azure credentials are correct

### Deployment Issues  
- Check AKS cluster permissions
- Verify resource group and cluster name are correct
- Ensure application listens on port 3000
- Check if health endpoints are implemented

### Branch Not Triggering
- Only `main` and `dev` branches trigger CI/CD
- Only `push` events trigger the pipeline in `main`
- Check branch name spelling

## Contributing

To modify the CI/CD pipeline:

1. Fork this repository
2. Make changes to workflow files
3. Test with a sample application
4. Submit pull request
