# Deployment Workflows - Usage Guide

Quick start guide for using the reusable deployment workflows.

## Quick Start

### Stage Deployment

```yaml
# .github/workflows/deploy.yaml
name: Deploy

on:
  push:
    branches: [development]

jobs:
  stage:
    uses: zopsmart/zs-workflows/.github/workflows/stage-deploy.yaml@main
    with:
      SVC_NAME: my-service
      BUILD_COMMAND: 'go build -o main ./cmd/...'
    secrets: inherit
```

### Production Deployment

```yaml
# .github/workflows/deploy.yaml
name: Deploy

on:
  release:
    types: [published]

jobs:
  prod:
    if: startsWith(github.ref, 'refs/tags/v')
    uses: zopsmart/zs-workflows/.github/workflows/prod-deploy.yaml@main
    with:
      SVC_NAME: my-service
    secrets: inherit
```

## Workflows

### stage-deploy.yaml

Builds, pushes, and deploys your application to a staging environment.

**When it runs:** Push to `development` branch

**What it does:**
1. Detects language from build command
2. Checks for code vs config-only changes
3. Builds application (Go/Node)
4. Builds and pushes Docker image
5. Deploys to Kubernetes cluster
6. Updates ConfigMap from env file

#### Required Inputs

| Input | Description |
|-------|-------------|
| `SVC_NAME` | Service name (Kubernetes deployment/cronjob name) |
| `BUILD_COMMAND` | Build command (e.g., `go build -o main ./cmd/...` or `yarn build`) |

#### Optional Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `LANGUAGE` | Auto-detected | Build language: `go`, `node`, or `generic` |
| `DOCKER_FILE_PATH` | `.` | Path to Dockerfile directory |
| `BUILD_ARGUMENTS` | | Docker build arguments |
| `GO_VERSION` | `1.20` | Go version (when `LANGUAGE=go`) |
| `NODE_VERSION` | `18` | Node version (when `LANGUAGE=node`) |
| `REGISTRY_TYPE` | Auto-detected | Registry provider: `gar`, `gcr`, `ecr`, `acr`, `dockerhub`, `ghcr`, `custom` |
| `IMAGE_REGISTRY` | From `vars.*` | Registry URL |
| `REGISTRY_PROJECT` | From `vars.*` | Project/namespace within registry |
| `REGISTRY_REPO` | From `vars.*` | Repository name within project |
| `CLUSTER_PROJECT` | From `vars.*` | GCP project (required for GKE) |
| `CLUSTER_NAME` | From `vars.*` | Kubernetes cluster name |
| `CLUSTER_REGION` | From `vars.*` | Cluster region |
| `AZURE_RESOURCE_GROUP` | From `vars.*` | Azure resource group (required for AKS) |
| `NAMESPACE` | `{APP_NAME}-stage` | Kubernetes namespace |
| `TYPE` | `deployment` | Workload type: `deployment` or `cron` |
| `DEPLOY_METHOD` | `kubectl` | Deploy method: `kubectl` or `helm` |
| `ENV_FILE_PATH` | `configs/.stage.env` | Path to env file (empty string skips configmap) |
| `REACT_APP` | `false` | Enable React-specific configmap format |

#### Required Secrets

| Secret | Description |
|--------|-------------|
| `REGISTRY_CREDENTIALS` | Registry auth (GCP SA JSON, AWS JSON, Azure JSON, or token) |
| `STAGE_CLUSTER_CREDENTIALS` | Cluster auth (GCP SA JSON, AWS JSON, Azure JSON, or base64 kubeconfig) |
| `PAT` | GitHub PAT for private dependencies (optional) |
| `NPM_TOKEN` | NPM token for private packages (optional, Node builds) |

#### Full Example

```yaml
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
      BUILD_COMMAND: 'go build -o main ./cmd/...'
      # REACT_APP: true  # Uncomment for React/frontend apps
    secrets: inherit
```

---

### prod-deploy.yaml

Retags the staging image and deploys to production.

**When it runs:** Version tag push (e.g., `v1.2.3`)

**What it does:**
1. Validates semantic version tag
2. Retags SHA image with release version
3. Deploys to production cluster
4. Updates ConfigMap from env file

#### Required Inputs

| Input | Description |
|-------|-------------|
| `SVC_NAME` | Service name (Kubernetes deployment/cronjob name) |

#### Optional Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `REGISTRY_TYPE` | Auto-detected | Registry provider |
| `IMAGE_REGISTRY` | From `vars.*` | Registry URL |
| `REGISTRY_PROJECT` | From `vars.*` | Project/namespace within registry |
| `REGISTRY_REPO` | From `vars.*` | Repository name within project |
| `CLUSTER_PROJECT` | From `vars.*` | GCP project (required for GKE) |
| `CLUSTER_NAME` | From `vars.*` | Kubernetes cluster name |
| `CLUSTER_REGION` | From `vars.*` | Cluster region |
| `AZURE_RESOURCE_GROUP` | From `vars.*` | Azure resource group (required for AKS) |
| `NAMESPACE` | `{APP_NAME}` | Kubernetes namespace |
| `TYPE` | `deployment` | Workload type: `deployment` or `cron` |
| `DEPLOY_METHOD` | `kubectl` | Deploy method: `kubectl` or `helm` |
| `ENV_FILE_PATH` | `configs/.prod.env` | Path to env file |
| `REACT_APP` | `false` | Enable React-specific configmap format |

#### Required Secrets

| Secret | Description |
|--------|-------------|
| `REGISTRY_CREDENTIALS` | Registry auth (GCP SA JSON, AWS JSON, Azure JSON, or token) |
| `PROD_CLUSTER_CREDENTIALS` | Cluster auth (GCP SA JSON, AWS JSON, Azure JSON, or base64 kubeconfig) |
| `PAT` | GitHub PAT for Helm repo access (optional) |

#### Full Example

```yaml
jobs:
  prod:
    if: startsWith(github.ref, 'refs/tags/v')
    uses: zopsmart/zs-workflows/.github/workflows/prod-deploy.yaml@main
    with:
      SVC_NAME: my-service
      # REACT_APP: true  # Uncomment for React/frontend apps
    secrets: inherit
```

---

## Composite Actions

For standalone use outside the main workflows.

### generate-configmap

Generates a Kubernetes ConfigMap YAML file from an environment file.

**Purpose:** Create ConfigMap manifests without applying them.

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `svc_name` | Yes | | Service/ConfigMap name |
| `env_file_path` | Yes | | Path to env file |
| `react_app` | No | `false` | Enable React-specific format |

| Output | Description |
|--------|-------------|
| `configmap_file` | Path to generated ConfigMap YAML |

```yaml
- uses: zopsmart/zs-workflows/.github/actions/generate-configmap@main
  with:
    svc_name: my-service
    env_file_path: ./configs/stage.env
```

See [generate-configmap README](../.github/actions/generate-configmap/README.md) for details.

---

### apply-configmap

Downloads a ConfigMap artifact and applies it to a Kubernetes cluster.

**Purpose:** Apply ConfigMaps from artifacts in multi-job workflows.

| Input | Required | Description |
|-------|----------|-------------|
| `artifact_name` | Yes | Name of artifact containing ConfigMap YAML |
| `namespace` | Yes | Kubernetes namespace |

```yaml
- uses: zopsmart/zs-workflows/.github/actions/apply-configmap@main
  with:
    artifact_name: configmap-my-service
    namespace: production
```

**Prerequisites:** kubectl must be authenticated to the cluster.

See [apply-configmap README](../.github/actions/apply-configmap/README.md) for details.

---

### validate-tag

Validates that a semantic version tag is the latest and ahead of previous tags.

**Purpose:** Prevent deploying stale or incorrectly versioned releases.

| Input | Required | Description |
|-------|----------|-------------|
| `tag` | Yes | Tag to validate (e.g., `v1.2.3`) |

| Output | Description |
|--------|-------------|
| `valid` | `true` if tag passes validation |
| `previous_tag` | Previous tag used for comparison |
| `error` | Error message if validation failed |

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0  # Required for tag history

- uses: zopsmart/zs-workflows/.github/actions/validate-tag@main
  id: validate
  with:
    tag: ${{ github.ref_name }}

- name: Fail if invalid
  if: steps.validate.outputs.valid != 'true'
  run: |
    echo "::error::${{ steps.validate.outputs.error }}"
    exit 1
```

See [validate-tag README](../.github/actions/validate-tag/README.md) for details.

---

### cluster-auth

Authenticates to Kubernetes clusters with auto-detection of auth method.

**Purpose:** Unified authentication for GKE, EKS, AKS, and kubeconfig.

| Input | Required | Description |
|-------|----------|-------------|
| `environment` | Yes | Environment name (stage, prod) |
| `credentials` | Yes | Cluster credentials (GCP SA JSON, AWS JSON, Azure JSON, or base64 kubeconfig) |
| `cluster_name` | Conditional | Cluster name (required for GKE/EKS/AKS) |
| `cluster_region` | Conditional | Cluster region (required for GKE/EKS) |
| `cluster_project` | Conditional | GCP project (required for GKE) |
| `azure_resource_group` | Conditional | Azure resource group (required for AKS) |

| Output | Description |
|--------|-------------|
| `auth_method` | Detected auth method: `gke`, `eks`, `aks`, `kubeconfig` |
| `kubeconfig_path` | Path to kubeconfig file |

**GKE:**
```yaml
- uses: zopsmart/zs-workflows/.github/actions/cluster-auth@main
  with:
    environment: stage
    credentials: ${{ secrets.CLUSTER_CREDENTIALS }}
    cluster_name: my-cluster
    cluster_region: us-central1
    cluster_project: my-gcp-project
```

**EKS:**
```yaml
- uses: zopsmart/zs-workflows/.github/actions/cluster-auth@main
  with:
    environment: stage
    credentials: ${{ secrets.CLUSTER_CREDENTIALS }}
    cluster_name: my-eks-cluster
    cluster_region: us-east-1
```

**AKS:**
```yaml
- uses: zopsmart/zs-workflows/.github/actions/cluster-auth@main
  with:
    environment: stage
    credentials: ${{ secrets.CLUSTER_CREDENTIALS }}
    cluster_name: my-aks-cluster
    azure_resource_group: my-resource-group
```

**Kubeconfig:**
```yaml
- uses: zopsmart/zs-workflows/.github/actions/cluster-auth@main
  with:
    environment: stage
    credentials: ${{ secrets.CLUSTER_CREDENTIALS }}  # base64-encoded kubeconfig
```

---

## Configuration

### Repository Variables

Set in Settings > Secrets and variables > Actions > Variables:

| Variable | Description | Example |
|----------|-------------|---------|
| `IMAGE_REGISTRY` | Registry URL | `us-docker.pkg.dev` |
| `REGISTRY_PROJECT` | Project/namespace | `my-gcp-project` |
| `REGISTRY_REPO` | Repository name | `docker-registry` |
| `CLUSTER_PROJECT` | GCP project (GKE) | `my-gcp-project` |
| `CLUSTER_NAME` | Cluster name | `my-cluster` |
| `CLUSTER_REGION` | Cluster region | `us-central1` |
| `AZURE_RESOURCE_GROUP` | Azure RG (AKS) | `my-resource-group` |
| `APP_NAME` | Application name | `my-service` |
| `STAGE_NAMESPACE` | Staging namespace | `my-app-stage` |
| `PROD_NAMESPACE` | Production namespace | `my-app` |

### Repository Secrets

Set in Settings > Secrets and variables > Actions > Secrets:

| Secret | Description |
|--------|-------------|
| `REGISTRY_CREDENTIALS` | Registry auth (GCP SA JSON, AWS JSON, Azure JSON, or token) |
| `STAGE_CLUSTER_CREDENTIALS` | Stage cluster auth |
| `PROD_CLUSTER_CREDENTIALS` | Prod cluster auth |
| `CLUSTER_CREDENTIALS` | Fallback cluster auth (used if env-specific not provided) |
| `PAT` | GitHub PAT for private dependencies |
| `NPM_TOKEN` | NPM token for private packages |

---

## Common Patterns

### ConfigMap-only Updates

When only the env file changes (no code changes), the workflow updates only the ConfigMap without rebuilding or redeploying:

```
Push with config changes only → check-changes detects → update-configmap-only job runs
```

### Helm Deployments

```yaml
jobs:
  stage:
    uses: zopsmart/zs-workflows/.github/workflows/stage-deploy.yaml@main
    with:
      SVC_NAME: my-service
      BUILD_COMMAND: 'go build -o main ./cmd/...'
      DEPLOY_METHOD: helm
      HELM_VALUES_PATH: ./helm/values-stage.yaml
    secrets: inherit
```

### Multi-environment Setup

Use environment-specific secrets for different clusters:

```yaml
# Repository secrets:
# STAGE_CLUSTER_CREDENTIALS - staging cluster auth
# PROD_CLUSTER_CREDENTIALS - production cluster auth

jobs:
  stage:
    uses: zopsmart/zs-workflows/.github/workflows/stage-deploy.yaml@main
    # Uses STAGE_CLUSTER_CREDENTIALS automatically
    secrets: inherit

  prod:
    uses: zopsmart/zs-workflows/.github/workflows/prod-deploy.yaml@main
    # Uses PROD_CLUSTER_CREDENTIALS automatically
    secrets: inherit
```

### React/Frontend Apps

```yaml
jobs:
  stage:
    uses: zopsmart/zs-workflows/.github/workflows/stage-deploy.yaml@main
    with:
      SVC_NAME: my-frontend
      BUILD_COMMAND: 'yarn build'
      REACT_APP: true  # Generates browser-compatible ConfigMap
    secrets: inherit
```
