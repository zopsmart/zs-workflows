# Deployment Workflow Examples

Reusable GitHub Actions workflows for building and deploying containerized applications to Kubernetes. Supports multiple container registries and cloud providers.

## Quick Start

```yaml
# .github/workflows/deploy.yaml
name: Deploy

on:
  push:
    branches: [development]
  release:
    types: [published]

jobs:
  stage:
    if: github.ref == 'refs/heads/development'
    uses: zopsmart/zs-workflows/.github/workflows/stage-deploy.yaml@main
    with:
      SVC_NAME: my-service
      BUILD_COMMAND: go build -o main ./cmd/...  # Language auto-detected
    secrets: inherit

  prod:
    if: startsWith(github.ref, 'refs/tags/v')
    uses: zopsmart/zs-workflows/.github/workflows/prod-deploy.yaml@main
    with:
      SVC_NAME: my-service
    secrets: inherit
```

## Supported Registries

| Registry Type | `REGISTRY_TYPE` | Default `IMAGE_REGISTRY` |
|---------------|-----------------|--------------------------|
| Google Artifact Registry | `gar` | _(required)_ |
| Google Container Registry | `gcr` | `gcr.io` |
| Amazon ECR | `ecr` | _(required)_ |
| Azure Container Registry | `acr` | _(required)_ |
| Docker Hub | `dockerhub` | `docker.io` |
| GitHub Container Registry | `ghcr` | `ghcr.io` |
| Custom Registry | `custom` | _(required)_ |

## Supported Cluster Auth Methods

| Auth Method | Required Inputs | Credentials Format |
|-------------|-----------------|-------------------|
| Google Kubernetes Engine | `CLUSTER_PROJECT`, `CLUSTER_NAME`, `CLUSTER_REGION` | GCP SA JSON |
| Amazon EKS | `CLUSTER_NAME`, `AWS_REGION` | AWS JSON |
| Azure AKS | `CLUSTER_NAME`, `AZURE_RESOURCE_GROUP` | Azure JSON |
| Kubeconfig | _(none)_ | Base64-encoded kubeconfig |

**Note:** Credential types are auto-detected from the JSON structure of `CLUSTER_CREDENTIALS`.

## Examples

### Go Services

| Example | Registry | Cluster |
|---------|----------|---------|
| [go/gar-gke.yaml](go/gar-gke.yaml) | Google Artifact Registry | GKE |
| [go/ecr-eks.yaml](go/ecr-eks.yaml) | Amazon ECR | EKS |
| [go/acr-aks.yaml](go/acr-aks.yaml) | Azure Container Registry | AKS |
| [go/ghcr-kubeconfig.yaml](go/ghcr-kubeconfig.yaml) | GitHub Container Registry | Kubeconfig |
| [go/dockerhub.yaml](go/dockerhub.yaml) | Docker Hub | Kubeconfig |

### Node/Yarn Services

| Example | Registry | Cluster |
|---------|----------|---------|
| [node/gar-gke.yaml](node/gar-gke.yaml) | Google Artifact Registry | GKE |
| [node/ecr-eks.yaml](node/ecr-eks.yaml) | Amazon ECR | EKS |
| [node/acr-aks.yaml](node/acr-aks.yaml) | Azure Container Registry | AKS |
| [node/ghcr-kubeconfig.yaml](node/ghcr-kubeconfig.yaml) | GitHub Container Registry | Kubeconfig |
| [node/dockerhub.yaml](node/dockerhub.yaml) | Docker Hub | Kubeconfig |

## Required Secrets

| Secret | Description |
|--------|-------------|
| `REGISTRY_CREDENTIALS` | Registry auth (GCP SA JSON, AWS JSON, Azure JSON, or token) |
| `CLUSTER_CREDENTIALS` | Cluster auth (GCP SA JSON, AWS JSON, Azure JSON, or base64 kubeconfig) |
| `PAT` | GitHub PAT for private dependencies |
| `NPM_TOKEN` | NPM token for private packages (Node projects) |

For detailed configuration options, see [docs/DEPLOYMENT-WORKFLOWS.md](../docs/DEPLOYMENT-WORKFLOWS.md).
