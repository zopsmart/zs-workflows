# Deployment Workflows - Technical Reference

This document describes the internal structure and configuration of the deployment workflows.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              CONSUMER REPO WORKFLOW                              │
│  ┌───────────────┐   ┌───────────────┐   ┌───────────────┐                      │
│  │  PR Created   │   │ Push to dev   │   │ Release Tag   │                      │
│  │  (tests only) │   │ (stage deploy)│   │ (prod deploy) │                      │
│  └───────┬───────┘   └───────┬───────┘   └───────┬───────┘                      │
└──────────┼───────────────────┼───────────────────┼──────────────────────────────┘
           │                   │                   │
           ▼                   ▼                   ▼
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│   _test-go.yaml  │  │ stage-deploy.yaml│  │ prod-deploy.yaml │
│  _test-node.yaml │  │                  │  │                  │
└──────────────────┘  └────────┬─────────┘  └────────┬─────────┘
                               │                     │
                               ▼                     ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                            STAGE DEPLOY FLOW                                      │
│                                                                                   │
│  ┌──────────────────┐                                                            │
│  │ resolve-config   │  Detect language, resolve registry/cluster vars            │
│  └────────┬─────────┘                                                            │
│           │                                                                       │
│           ▼                                                                       │
│  ┌──────────────────┐                                                            │
│  │ _check-changes   │  Detect: code changes vs env file changes                  │
│  └────────┬─────────┘                                                            │
│           │                                                                       │
│           ├─── code changed? ───┐                                                │
│           │                     ▼                                                │
│           │          ┌──────────────────┐                                        │
│           │          │   BUILD PHASE    │                                        │
│           │          │  ┌────────────┐  │                                        │
│           │          │  │  _build    │  │  LANGUAGE=go: go mod download → build │
│           │          │  │            │  │  LANGUAGE=node: yarn install → build  │
│           │          │  │            │  │  LANGUAGE=generic: build only         │
│           │          │  └─────┬──────┘  │                                        │
│           │          │        │         │                                        │
│           │          │        ▼         │                                        │
│           │          │  ┌────────────┐  │                                        │
│           │          │  │docker-build│  │  Multi-registry: GAR/GCR/ECR/ACR/     │
│           │          │  │   -push    │  │  DockerHub/GHCR/Custom                │
│           │          │  └─────┬──────┘  │                                        │
│           │          └────────┼─────────┘                                        │
│           │                   │                                                   │
│           │                   ▼                                                   │
│           │          ┌──────────────────┐                                        │
│           │          │    _deploy       │  Multi-cloud: GKE/EKS/AKS/kubeconfig  │
│           │          └────────┬─────────┘                                        │
│           │                   │                                                   │
│           ├───────────────────┤                                                   │
│           │                   ▼                                                   │
│           │          ┌──────────────────┐                                        │
│           └─────────►│_update-configmap │  Update K8s ConfigMap from env file   │
│      (env changed)   └──────────────────┘                                        │
│                                                                                   │
└──────────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────────┐
│                             PROD DEPLOY FLOW                                      │
│                                                                                   │
│  ┌──────────────────┐                                                            │
│  │ resolve-config   │  Extract release version, resolve vars                     │
│  └────────┬─────────┘                                                            │
│           │                                                                       │
│           ▼                                                                       │
│  ┌──────────────────┐                                                            │
│  │  _retag-image    │  Retag SHA image → release version (v1.0.0)               │
│  └────────┬─────────┘                                                            │
│           │                                                                       │
│           ▼                                                                       │
│  ┌──────────────────┐                                                            │
│  │    _deploy       │  Deploy to production cluster                              │
│  └────────┬─────────┘                                                            │
│           │                                                                       │
│           ▼                                                                       │
│  ┌──────────────────┐                                                            │
│  │_update-configmap │  Update prod ConfigMap                                     │
│  └──────────────────┘                                                            │
│                                                                                   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

## Credential Resolution

Credentials support both full JSON and partial/split configurations. User inputs always have the highest priority.

### Resolution Priority Order

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CREDENTIAL RESOLUTION ORDER                       │
│                                                                      │
│  ┌──────────────┐                                                   │
│  │  1. INPUT    │ ◄── Workflow input parameter (highest priority)   │
│  └──────┬───────┘                                                   │
│         │ (if empty)                                                │
│         ▼                                                           │
│  ┌──────────────┐                                                   │
│  │  2. JSON     │ ◄── Key from CLUSTER_CREDENTIALS JSON             │
│  └──────┬───────┘                                                   │
│         │ (if empty)                                                │
│         ▼                                                           │
│  ┌──────────────┐                                                   │
│  │  3. VARS     │ ◄── Repository variable (vars.*)                  │
│  └──────┬───────┘                                                   │
│         │ (if empty)                                                │
│         ▼                                                           │
│  ┌──────────────┐                                                   │
│  │  4. SECRETS  │ ◄── Repository secret (secrets.*)                 │
│  └──────────────┘                                                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### EKS Credential Resolution

```
┌─────────────────────────────────────────────────────────────────────┐
│                      EKS CREDENTIAL RESOLUTION                       │
│                                                                      │
│  aws_secret_access_key:                                             │
│  ├─► CLUSTER_CREDENTIALS (if JSON with aws_secret_access_key key)   │
│  ├─► CLUSTER_CREDENTIALS (if raw string, use as-is)                 │
│  └─► secrets.AWS_SECRET_ACCESS_KEY                                  │
│                                                                      │
│  aws_access_key_id:                                                 │
│  ├─► inputs.AWS_ACCESS_KEY_ID        (highest priority)             │
│  ├─► CLUSTER_CREDENTIALS JSON key                                   │
│  ├─► vars.AWS_ACCESS_KEY_ID                                         │
│  └─► secrets.AWS_ACCESS_KEY_ID                                      │
│                                                                      │
│  aws_region:                                                        │
│  ├─► inputs.AWS_REGION               (highest priority)             │
│  ├─► CLUSTER_CREDENTIALS JSON key                                   │
│  ├─► vars.AWS_REGION                                                │
│  └─► inputs.CLUSTER_REGION                                          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### AKS Credential Resolution

```
┌─────────────────────────────────────────────────────────────────────┐
│                      AKS CREDENTIAL RESOLUTION                       │
│                                                                      │
│  clientSecret:                                                      │
│  ├─► CLUSTER_CREDENTIALS (if JSON with clientSecret key)            │
│  ├─► CLUSTER_CREDENTIALS (if raw string, use as-is)                 │
│  └─► secrets.AZURE_CLIENT_SECRET                                    │
│                                                                      │
│  clientId:                                                          │
│  ├─► inputs.AZURE_CLIENT_ID          (highest priority)             │
│  ├─► CLUSTER_CREDENTIALS JSON key                                   │
│  ├─► vars.AZURE_CLIENT_ID                                           │
│  └─► secrets.AZURE_CLIENT_ID                                        │
│                                                                      │
│  tenantId:                                                          │
│  ├─► inputs.AZURE_TENANT_ID          (highest priority)             │
│  ├─► CLUSTER_CREDENTIALS JSON key                                   │
│  ├─► vars.AZURE_TENANT_ID                                           │
│  └─► secrets.AZURE_TENANT_ID                                        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Environment-Specific Cluster Credentials

```
┌─────────────────────────────────────────────────────────────────────┐
│              CLUSTER CREDENTIALS FALLBACK CHAIN                      │
│                                                                      │
│  stage-deploy.yaml:                                                 │
│  ├─► secrets.STAGE_CLUSTER_CREDENTIALS                              │
│  ├─► secrets.CLUSTER_CREDENTIALS                                    │
│  ├─► secrets.STAGE_DEPLOY_KEY        (legacy)                       │
│  └─► secrets.DEPLOY_KEY              (legacy)                       │
│                                                                      │
│  prod-deploy.yaml:                                                  │
│  ├─► secrets.PROD_CLUSTER_CREDENTIALS                               │
│  ├─► secrets.CLUSTER_CREDENTIALS                                    │
│  ├─► secrets.PROD_DEPLOY_KEY         (legacy)                       │
│  └─► secrets.DEPLOY_KEY              (legacy)                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Example: Split Credentials Configuration

```yaml
# Consumer workflow using split credentials
jobs:
  deploy:
    uses: zopsmart/zs-workflows/.github/workflows/stage-deploy.yaml@main
    with:
      SVC_NAME: my-service
      BUILD_COMMAND: 'go build -o main ./cmd/...'
      # Override access key ID via input (highest priority)
      AWS_ACCESS_KEY_ID: 'AKIAIOSFODNN7EXAMPLE'
    secrets:
      # Only the secret key in CLUSTER_CREDENTIALS
      STAGE_CLUSTER_CREDENTIALS: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      # Other values from vars.* or secrets.* automatically resolved

# Repository variables (vars.*)
# AWS_REGION: us-east-1

# Repository secrets (secrets.*)
# AWS_SECRET_ACCESS_KEY: wJalrXUtnFEMI/...
```

## Workflows

### Entry Points

| Workflow | Description |
|----------|-------------|
| `stage-deploy.yaml` | Build app (Go or Node), push image, deploy to staging |
| `prod-deploy.yaml` | Retag staging image, deploy to production |

### Building Blocks

| Workflow | Description |
|----------|-------------|
| `_build.yaml` | Build app (Go/Node/generic) and push Docker image |
| `_retag-image.yaml` | Retag existing image with new tag |
| `_deploy.yaml` | Deploy image to Kubernetes |
| `_update-configmap.yaml` | Update Kubernetes ConfigMap from env file |
| `_check-changes.yaml` | Detect code vs config changes |
| `_resolve-registry-config.yaml` | Resolve registry type and URL from inputs/vars |
| `_resolve-deploy-config.yaml` | Resolve cluster/namespace config from inputs/vars |

### Composite Actions

| Action | Description |
|--------|-------------|
| `docker-build-push` | Build and push Docker image to any registry |
| `registry-login` | Authenticate to container registry (GAR/GCR/ECR/ACR/DockerHub/GHCR) |
| `cluster-auth` | Authenticate to Kubernetes cluster (GKE/EKS/AKS/kubeconfig) |
| `resolve-image-path` | Resolve registry URL and construct image path |

## Language Auto-Detection

The `LANGUAGE` input is optional. The workflow automatically detects the language from `BUILD_COMMAND`:

| BUILD_COMMAND Pattern | Detected Language |
|----------------------|-------------------|
| Contains `go build`, `go run`, `go test`, `go install` | `go` |
| Contains `yarn` or `npm` | `node` |
| Other | `generic` (skips dependency setup) |

**Examples:**
- `go build -o main ./cmd/...` → Detected as `go`
- `yarn build` → Detected as `node`
- `npm run build` → Detected as `node`
- `make build` → Detected as `generic` (warning logged, skips `go mod download`/`yarn install`)

If language cannot be detected, the workflow logs a warning and continues without language-specific dependency initialization. The Docker build still runs normally, so you can handle dependencies in your Dockerfile.

## Configuration

### Repository Variables (`vars.*`)

Set these in your repository's Settings > Secrets and variables > Actions > Variables:

| Variable | Description | Example |
|----------|-------------|---------|
| `REGISTRY_TYPE` | Registry provider. Auto-detected from IMAGE_REGISTRY URL. Only needed for gcr/ghcr/dockerhub | `gar`, `ecr`, `ghcr` |
| `IMAGE_REGISTRY` | Registry URL. REGISTRY_TYPE auto-detected from URL pattern | `us-docker.pkg.dev` |
| `REGISTRY_PROJECT` | Project/namespace. Used for image path construction | `my-gcp-project` |
| `REGISTRY_REPO` | Repository name. Used for image path construction | `docker-registry` |
| `CLUSTER_PROJECT` | GCP project. **Required for GKE auth** | `my-gcp-project` |
| `CLUSTER_NAME` | Cluster name. **Required for GKE/EKS/AKS auth** | `my-cluster` |
| `CLUSTER_REGION` | Cluster region. **Required for GKE/EKS auth** (default: us-central1). Also checks `vars.AWS_REGION` as fallback | `us-central1` |
| `AWS_REGION` | Legacy: Fallback for `CLUSTER_REGION` when using EKS | `us-east-1` |
| `AZURE_RESOURCE_GROUP` | Azure RG. **Required for AKS auth** | `my-resource-group` |
| `APP_NAME` | Application name. Used as fallback for namespace and repo | `my-service` |
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
3. Derived default (based on `REGISTRY_TYPE`)

### Auth Method Auto-Detection

The cluster auth method is automatically detected (no input required):

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AUTH METHOD AUTO-DETECTION                        │
│                                                                      │
│  1. From JSON credentials:                                          │
│     ├─► type: service_account           → GKE                       │
│     ├─► aws_access_key_id present       → EKS                       │
│     ├─► clientId present                → AKS                       │
│     └─► Base64 kubeconfig               → kubeconfig                │
│                                                                      │
│  2. From inputs/vars/secrets (if JSON detection fails):             │
│     ├─► AWS_ACCESS_KEY_ID or AWS_SECRET_ACCESS_KEY set   → EKS      │
│     ├─► AZURE_CLIENT_ID or AZURE_CLIENT_SECRET set       → AKS      │
│     └─► CLUSTER_PROJECT set                              → GKE      │
│                                                                      │
│  3. Default: kubeconfig                                             │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

## Registry Type Auto-Detection

`REGISTRY_TYPE` is automatically detected from `IMAGE_REGISTRY` URL patterns:

| IMAGE_REGISTRY Pattern | Detected Type |
|------------------------|---------------|
| `*.dkr.ecr.*.amazonaws.com` | `ecr` |
| `*.azurecr.io` | `acr` |
| `*-docker.pkg.dev` | `gar` |
| `gcr.io`, `*.gcr.io` | `gcr` |
| `ghcr.io` | `ghcr` |
| `docker.io`, `registry.hub.docker.com` | `dockerhub` |
| Other | `custom` |

**Usage:**
- Provide just `IMAGE_REGISTRY` and the type is auto-detected
- For `gcr`, `ghcr`, `dockerhub`: can provide just `REGISTRY_TYPE` (URL defaults automatically)
- If both provided: they must be consistent or workflow errors

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

### Required Field Types

Fields can be required in different ways:

| Requirement Type | Meaning |
|-----------------|---------|
| **Yes** | Must be provided as workflow input |
| **Yes*** | Required, but can be set via workflow input OR repository variable (`vars.*`). Workflow errors if neither is provided |
| **Conditional** | Required only when using specific providers (e.g., `IMAGE_REGISTRY` is required for gar/ecr/acr/custom but auto-defaults for gcr/dockerhub/ghcr) |
| **No** | Optional, uses default or falls back to repository variable |

### stage-deploy.yaml

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `SVC_NAME` | Yes | | Service name |
| `BUILD_COMMAND` | Yes | | Build command |
| `LANGUAGE` | No | Auto-detected | `go` or `node` (auto-detected from BUILD_COMMAND) |
| `REGISTRY_TYPE` | Auto | `vars.REGISTRY_TYPE` | Registry provider. **Auto-detected from IMAGE_REGISTRY URL**. Only needed for gcr/ghcr/dockerhub without explicit URL |
| `IMAGE_REGISTRY` | Conditional | `vars.IMAGE_REGISTRY` | Registry URL. REGISTRY_TYPE auto-detected from URL pattern. **Required for gar/ecr/acr/custom** |
| `REGISTRY_PROJECT` | No | `vars.REGISTRY_PROJECT` | Project/namespace. Used for image path construction |
| `REGISTRY_REPO` | No | `vars.REGISTRY_REPO` | Repository name. Used for image path construction |
| `CLUSTER_PROJECT` | Conditional | `vars.CLUSTER_PROJECT` | GCP project. **Required for GKE auth** |
| `CLUSTER_NAME` | Conditional | `vars.CLUSTER_NAME` | Cluster name. **Required for GKE/EKS/AKS auth** |
| `CLUSTER_REGION` | Conditional | `us-central1` | Cluster region. **Required for GKE/EKS auth**. Falls back to `vars.CLUSTER_REGION`, then `vars.AWS_REGION` |
| `AZURE_RESOURCE_GROUP` | Conditional | `vars.AZURE_RESOURCE_GROUP` | Azure RG. **Required for AKS auth** |
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
| `REGISTRY_TYPE` | Auto | `vars.REGISTRY_TYPE` | Registry provider. **Auto-detected from IMAGE_REGISTRY URL**. Only needed for gcr/ghcr/dockerhub without explicit URL |
| `IMAGE_REGISTRY` | Conditional | `vars.IMAGE_REGISTRY` | Registry URL. REGISTRY_TYPE auto-detected from URL pattern. **Required for gar/ecr/acr/custom** |
| `REGISTRY_PROJECT` | No | `vars.REGISTRY_PROJECT` | Project/namespace. Used for image path construction |
| `REGISTRY_REPO` | No | `vars.REGISTRY_REPO` | Repository name. Used for image path construction |
| `CLUSTER_PROJECT` | Conditional | `vars.CLUSTER_PROJECT` | GCP project. **Required for GKE auth** |
| `CLUSTER_NAME` | Conditional | `vars.CLUSTER_NAME` | Cluster name. **Required for GKE/EKS/AKS auth** |
| `CLUSTER_REGION` | Conditional | `us-central1` | Cluster region. **Required for GKE/EKS auth**. Falls back to `vars.CLUSTER_REGION`, then `vars.AWS_REGION` |
| `AZURE_RESOURCE_GROUP` | Conditional | `vars.AZURE_RESOURCE_GROUP` | Azure RG. **Required for AKS auth** |
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
- `CLUSTER_CREDENTIALS` falls back to `STAGE_DEPLOY_KEY` / `PROD_DEPLOY_KEY` / `DEPLOY_KEY`
- `REGISTRY_PROJECT` falls back to `vars.GAR_PROJECT`
- `REGISTRY_REPO` falls back to `vars.GAR_REGISTRY`

To migrate, you **must** add `REGISTRY_TYPE` to your repository variables (it's required):

```bash
# Add to repository variables (REQUIRED)
REGISTRY_TYPE=gar
```

### Full Migration Steps

1. **Add required repository variable:**
   ```
   REGISTRY_TYPE=gar  # REQUIRED - workflow will fail without this
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
