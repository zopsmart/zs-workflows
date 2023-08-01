# App Template
This repository contains application deployment templates along with required helm values

### Variables(values.yaml)

| Inputs               | Type      | Required/Optional | <div style="width:420px">Description</div>                                                | Default                |
|----------------------|-----------|-------------------|-------------------------------------------------------------------------------------------|------------------------|
| maxCPU               | string    | Optional          | Maximum CPU resources                                                                     | `"500m"`               |
| maxMemory            | string    | Optional          | Maximum Memory resources                                                                  | `"512Mi"`              |
| maxReplicas          | number    | Optional          | Maximum replicas                                                                          | `4`                    |
| minCPU               | string    | Optional          | Minimum CPU resources                                                                     | `"250m"`               |
| minMemory            | string    | Optional          | Minimum Memory resources                                                                  | `"128Mi"`              |
| minReplicas          | number    | Optional          | Minimum replicas                                                                          | `2`                    |
| name                 | string    | Required          | Name of the service to be deployed(should be same as service provided at namespace level) | `"hello-api"`          |

#### env variables can be set under env: 
<pre>
env:
    HTTP_PORT: 8000
</pre>

### Workflow env variables
These variables are set by Terraform Framework as Github Action Variables.

| Inputs                   | Type         | Required/Optional | <div style="width:400px">Description</div>                | Default |
|--------------------------|--------------|-------------------|-----------------------------------------------------------|---------|
| app_name                 | string       | Required          | Name of the service                                       | nil     |
| namespace                | string       | Required          | Name of the namespace in which the service to be deployed | nil     |
| cluster_name             | string       | Required          | Name of the Cluster                                       | nil     |
| cluster_project_name     | string       | Required          | Name of the GCP project                                   | nil     |
| gar_project_name         | string       | Required          | Name of the Google Artifact Registry project              | nil     |
| registry_name            | string       | Required          | Name of the Google Artifact Registry                      | nil     |

#### Workflow Secrets
These variables are set by Terraform Framework as Github Action Secrets.

| Secret      | Description                                                                                                   |
|-------------|---------------------------------------------------------------------------------------------------------------|
| PAT         | Github PAT with required permissions                                                                          |
| DEPLOY_KEY  | GCP service account credentials to deploy the services with the respective deployment environment as a prefix |
