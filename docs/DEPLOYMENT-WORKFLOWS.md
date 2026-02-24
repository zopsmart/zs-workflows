# Deployment Workflows - Technical Reference

This document describes the internal structure and configuration of the deployment workflows.

## Workflows

### Entry Points

| Workflow | Description |
|----------|-------------|
| `stage-deploy.yaml` | Build app (Go or Node), push image, deploy to staging |
| `prod-deploy.yaml` | Retag staging image, deploy to production |

### Building Blocks

| Workflow | Description |
|----------|-------------|
| `_build-go.yaml` | Build Go app and push Docker image |
| `_build-yarn.yaml` | Build Node/Yarn app and push Docker image |
| `_build-generic.yaml` | Build any app without language-specific setup |
| `_retag-image.yaml` | Retag existing image with new tag |
| `_deploy.yaml` | Deploy image to Kubernetes |
| `_update-configmap.yaml` | Update Kubernetes ConfigMap from env file |
| `_check-changes.yaml` | Detect code vs config changes |
| `_validate-cluster-auth.yaml` | Validate and authenticate to cluster |

## Language Auto-Detection

The `LANGUAGE` input is optional. The workflow automatically detects the language from `BUILD_COMMAND`:

| BUILD_COMMAND Pattern | Detected Language |
|----------------------|-------------------|
| Contains `go build`, `go run`, `go test`, `go install` | `go` |
| Contains `yarn` or `npm` | `node` |
| Other | `unknown` (skips dependency setup) |

**Examples:**
- `go build -o main ./cmd/...` → Detected as `go`
- `yarn build` → Detected as `node`
- `npm run build` → Detected as `node`
- `make build` → Detected as `unknown` (warning logged, skips `go mod download`/`yarn install`)

If language cannot be detected, the workflow logs a warning and continues without language-specific dependency initialization. The Docker build still runs normally, so you can handle dependencies in your Dockerfile.

## Configuration

### Repository Variables (`vars.*`)

Set these in your repository's Settings > Secrets and variables > Actions > Variables:

| Variable | Description | Example |
|----------|-------------|---------|
| `REGISTRY_TYPE` | Registry provider | `gar`, `ecr`, `ghcr` |
| `IMAGE_REGISTRY` | Registry URL | `us-docker.pkg.dev` |
| `REGISTRY_PROJECT` | Project/namespace | `my-gcp-project` |
| `REGISTRY_REPO` | Repository name | `docker-registry` |
| `CLUSTER_PROJECT` | GCP project (GKE) | `my-gcp-project` |
| `CLUSTER_NAME` | Cluster name | `my-cluster` |
| `CLUSTER_REGION` | Cluster region | `us-central1` |
| `AWS_REGION` | AWS region (EKS) | `us-east-1` |
| `AZURE_RESOURCE_GROUP` | Azure RG (AKS) | `my-resource-group` |
| `APP_NAME` | Application name | `my-service` |
| `STAGE_NAMESPACE` | Staging namespace | `my-app-stage` |
| `PROD_NAMESPACE` | Production namespace | `my-app` |

### Repository Secrets (`secrets.*`)

Set these in your repository's Settings > Secrets and variables > Actions > Secrets:

| Secret | Description |
|--------|-------------|
| `REGISTRY_CREDENTIALS` | Registry auth (auto-detected: GCP SA JSON, AWS JSON, Azure JSON, or password/token) |
| `CLUSTER_CREDENTIALS` | Cluster auth (auto-detected: GCP SA JSON, AWS JSON, Azure JSON, or base64 kubeconfig) |
| `PAT` | GitHub PAT for private dependencies |
| `NPM_TOKEN` | NPM token for private packages |

### Input Resolution Order

All inputs follow this resolution order:
1. Workflow input (if provided)
2. Repository variable (`vars.*`)
3. Derived default (based on `REGISTRY_TYPE` or `CLUSTER_AUTH_METHOD`)

## Image Path Construction

The image path is constructed based on which components are provided:

```
IMAGE_REGISTRY / REGISTRY_PROJECT / REGISTRY_REPO / SVC_NAME : TAG
```

| Components Provided | Result |
|---------------------|--------|
| All | `us-docker.pkg.dev/my-project/docker/api:v1` |
| Registry + Project | `gcr.io/my-project/api:v1` |
| Registry only | `docker.io/api:v1` |

### Examples by Registry

| Registry | IMAGE_REGISTRY | REGISTRY_PROJECT | REGISTRY_REPO | Result |
|----------|----------------|------------------|---------------|--------|
| GAR | `us-docker.pkg.dev` | `my-project` | `docker` | `us-docker.pkg.dev/my-project/docker/api:v1` |
| GCR | _(auto)_ | `my-project` | | `gcr.io/my-project/api:v1` |
| ECR | `123456.dkr.ecr.us-east-1.amazonaws.com` | | | `123456.dkr.ecr.us-east-1.amazonaws.com/api:v1` |
| ACR | `myregistry.azurecr.io` | | | `myregistry.azurecr.io/api:v1` |
| Docker Hub | _(auto)_ | `username` | | `docker.io/username/api:v1` |
| GHCR | _(auto)_ | `owner` | | `ghcr.io/owner/api:v1` |

## Workflow Inputs Reference

### stage-deploy.yaml

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `SVC_NAME` | Yes | | Service name |
| `BUILD_COMMAND` | Yes | | Build command |
| `LANGUAGE` | No | Auto-detected | `go` or `node` (auto-detected from BUILD_COMMAND) |
| `REGISTRY_TYPE` | No | `vars.REGISTRY_TYPE` | Registry provider |
| `IMAGE_REGISTRY` | No | Auto | Registry URL |
| `REGISTRY_PROJECT` | No | `vars.REGISTRY_PROJECT` | Project/namespace |
| `REGISTRY_REPO` | No | `vars.REGISTRY_REPO` | Repository name |
| `CLUSTER_PROJECT` | No | `vars.CLUSTER_PROJECT` | GCP project (GKE) |
| `CLUSTER_NAME` | No | `vars.CLUSTER_NAME` | Cluster name |
| `CLUSTER_REGION` | No | `us-central1` | Cluster region |
| `AWS_REGION` | No | `vars.AWS_REGION` | AWS region (EKS) |
| `AZURE_RESOURCE_GROUP` | No | `vars.AZURE_RESOURCE_GROUP` | Azure RG (AKS) |
| `NAMESPACE` | No | `{APP_NAME}-stage` | Kubernetes namespace |
| `TYPE` | No | `deployment` | `deployment` or `cron` |
| `DEPLOY_METHOD` | No | `kubectl` | `kubectl` or `helm` |
| `ENV_FILE_PATH` | No | `configs/.stage.env` | Env file path |
| `DOCKER_FILE_PATH` | No | `.` | Dockerfile directory |
| `BUILD_ARGUMENTS` | No | | Docker build args |
| `GO_VERSION` | No | `1.20` | Go version (for `LANGUAGE=go`) |
| `NODE_VERSION` | No | `18` | Node version (for `LANGUAGE=node`) |

### prod-deploy.yaml

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `SVC_NAME` | Yes | | Service name |
| `REGISTRY_TYPE` | No | `vars.REGISTRY_TYPE` | Registry provider |
| `IMAGE_REGISTRY` | No | Auto | Registry URL |
| `REGISTRY_PROJECT` | No | `vars.REGISTRY_PROJECT` | Project/namespace |
| `REGISTRY_REPO` | No | `vars.REGISTRY_REPO` | Repository name |
| `CLUSTER_PROJECT` | No | `vars.CLUSTER_PROJECT` | GCP project (GKE) |
| `CLUSTER_NAME` | No | `vars.CLUSTER_NAME` | Cluster name |
| `CLUSTER_REGION` | No | `us-central1` | Cluster region |
| `AWS_REGION` | No | `vars.AWS_REGION` | AWS region (EKS) |
| `AZURE_RESOURCE_GROUP` | No | `vars.AZURE_RESOURCE_GROUP` | Azure RG (AKS) |
| `NAMESPACE` | No | `{APP_NAME}` | Kubernetes namespace |
| `TYPE` | No | `deployment` | `deployment` or `cron` |
| `DEPLOY_METHOD` | No | `kubectl` | `kubectl` or `helm` |
| `ENV_FILE_PATH` | No | `configs/.prod.env` | Env file path |

## Credential Formats

### GCP Service Account JSON (GAR/GCR/GKE)

```json
{
  "type": "service_account",
  "project_id": "my-project",
  "private_key_id": "...",
  "private_key": "-----BEGIN PRIVATE KEY-----\n...",
  "client_email": "sa@my-project.iam.gserviceaccount.com"
}
```

### AWS Credentials (ECR/EKS)

```json
{
  "aws_access_key_id": "AKIAIOSFODNN7EXAMPLE",
  "aws_secret_access_key": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
}
```

### Azure Credentials (ACR/AKS)

```json
{
  "clientId": "12345678-1234-1234-1234-123456789abc",
  "clientSecret": "abc123~secret",
  "tenantId": "87654321-4321-4321-4321-cba987654321"
}
```

### Kubeconfig (Base64)

```bash
# Encode your kubeconfig
base64 -w 0 ~/.kube/config
```

## Migration from Legacy Workflows

If you're using the old GAR-specific workflows, here's how to migrate:

### Variable Mapping

| Old Variable | New Variable |
|--------------|--------------|
| `GAR_PROJECT` | `REGISTRY_PROJECT` |
| `GAR_REGISTRY` | `REGISTRY_REPO` |
| _(new)_ | `REGISTRY_TYPE` |

### Secret Mapping

| Old Secret | New Secret |
|------------|------------|
| `GAR_KEY` | `REGISTRY_CREDENTIALS` |
| `DEPLOY_KEY` | `CLUSTER_CREDENTIALS` |
| `STAGE_DEPLOY_KEY` | `CLUSTER_CREDENTIALS` |
| `PROD_DEPLOY_KEY` | `CLUSTER_CREDENTIALS` |

### Backwards Compatibility

The new workflows support legacy variable/secret names via fallback:
- `REGISTRY_CREDENTIALS` falls back to `GAR_KEY`
- `CLUSTER_CREDENTIALS` falls back to `STAGE_DEPLOY_KEY` / `PROD_DEPLOY_KEY` / `DEPLOY_KEY` / `GAR_KEY`
- `REGISTRY_PROJECT` falls back to `vars.GAR_PROJECT`
- `REGISTRY_REPO` falls back to `vars.GAR_REGISTRY`

To migrate, simply add `REGISTRY_TYPE` to your repository variables:

```bash
# Add to repository variables
REGISTRY_TYPE=gar
```

### Full Migration Steps

1. **Add new repository variable:**
   ```
   REGISTRY_TYPE=gar
   ```

2. **Optionally rename variables** (not required due to fallback):
   ```
   GAR_PROJECT -> REGISTRY_PROJECT
   GAR_REGISTRY -> REGISTRY_REPO
   ```

3. **Optionally rename secrets** (not required due to fallback):
   ```
   GAR_KEY -> REGISTRY_CREDENTIALS
   DEPLOY_KEY -> CLUSTER_CREDENTIALS
   ```

4. **Update workflow reference:**
   ```yaml
   # Old
   uses: zopsmart/zs-workflows/.github/workflows/go_gar_stage_deploy.yaml@main

   # New
   uses: zopsmart/zs-workflows/.github/workflows/stage-deploy.yaml@main
   ```
