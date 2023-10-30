# App Template
This repository contains reusable workflows and standard templates for `go` and `java`.

### Workflow env variables
These variables are set by Terraform Framework as GitHub Action Variables.

| Inputs                   | Type         | Required/Optional | <div style="width:400px">Description</div>                | Default |
|--------------------------|--------------|-------------------|-----------------------------------------------------------|---------|
| app_name                 | string       | Required          | Name of the service                                       | nil     |
| namespace                | string       | Required          | Name of the namespace in which the service to be deployed | nil     |
| cluster_name             | string       | Required          | Name of the Cluster                                       | nil     |
| cluster_project_name     | string       | Required          | Name of the GCP project                                   | nil     |
| gar_project_name         | string       | Required          | Name of the Google Artifact Registry project              | nil     |
| registry_name            | string       | Required          | Name of the Google Artifact Registry                      | nil     |

#### Workflow Secrets
These variables are set by Terraform Framework as GitHub Action Secrets.

| Secret      | Description                                                                                                   |
|-------------|---------------------------------------------------------------------------------------------------------------|
| PAT         | Github PAT with required permissions                                                                          |
| DEPLOY_KEY  | GCP service account credentials to deploy the services with the respective deployment environment as a prefix |
