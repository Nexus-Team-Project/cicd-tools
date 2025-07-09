# Example Application CI/CD Setup

This is an example of how to use the cicd-tools in your application repository.

## File: `.github/workflows/ci-cd.yml`

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, dev]

jobs:
  ci_cd:
    name: Build and Deploy
    uses: Nexus-Team-Project/cicd-tools/.github/workflows/main.yml@main
    secrets: inherit
```

## Required Repository Structure

```
your-app-repo/
├── .github/
│   └── workflows/
│       └── main.yml          # The file above
├── src/                       # Your application code
├── package.json               # If Node.js app
├── Dockerfile                 # Required for containerization
└── README.md
```

## Example Dockerfile (Node.js)

```dockerfile
FROM node:18-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy application code
COPY . .

# Expose port (must be 3000 for the K8s templates)
EXPOSE 3000

# Health check endpoint (recommended)
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/ || exit 1

# Start application
CMD ["npm", "start"]
```

## Example Health Endpoints (Express.js)

```javascript
// Add these to your Express app
app.get('/', (req, res) => {
  res.status(200).json({ status: 'healthy', timestamp: new Date().toISOString() });
});

app.get('/', (req, res) => {
  // Add any readiness checks here (database connectivity, etc.)
  res.status(200).json({ status: 'ready', timestamp: new Date().toISOString() });
});
```

## Required GitHub Secrets

Add these secrets to your repository settings:

1. Go to your repository → Settings → Secrets and variables → Actions
2. Add these repository secrets:
   - `AZURE_CLIENT_ID`
   - `AZURE_TENANT_ID` 
   - `AZURE_SUBSCRIPTION_ID`

## How It Works

1. **Push to `dev`**: Builds image with `vars.IMAGE_TAG_NP` tag, deploys to `vars.NAMESPACE_NAME_NP` namespace
2. **Push to `main`**: Builds image with `vars.IMAGE_TAG_PROD` tag, deploys to `vars.NAMESPACE_NAME_PROD` namespace
3. **Other branches**: No CI/CD runs

## Required GitHub Variables

Add these variables to your repository/organization settings:

1. Go to your repository → Settings → Secrets and variables → Actions → Variables tab
2. Add these repository variables:
   - `AZURE_CONTAINER_REGISTRY` - Name of your ACR (e.g., "myacr")
   - `RESOURCE_GROUP` - Resource group containing your ACR
   - `IMAGE_TAG_NP` - Image tag for non-production (e.g., "dev")
   - `IMAGE_TAG_PROD` - Image tag for production (e.g., "stable")
   - `NAMESPACE_NAME_NP` - Kubernetes namespace for non-production (e.g., "dev")
   - `NAMESPACE_NAME_PROD` - Kubernetes namespace for production (e.g., "production")

## Testing the Pipeline

1. Create a simple Node.js app with the health endpoints
2. Add the Dockerfile and workflow file
3. Set up the required GitHub variables
4. Push to `dev` branch
5. Check GitHub Actions tab for pipeline execution
6. Verify deployment in AKS: `kubectl get pods -n <your-np-namespace>`
