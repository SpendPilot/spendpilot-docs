# Azure Managed Identity Setup

The AKS stack uses:

- one user-assigned managed identity for application pods
- Azure Workload Identity federation from the AKS OIDC issuer
- a single Kubernetes service account annotation for the application namespace

Terraform provisions:

- `azurerm_user_assigned_identity`
- `azurerm_federated_identity_credential`
- role assignments for:
  - `Storage Blob Data Contributor`
  - `Cognitive Services User`
  - `Cognitive Services OpenAI User`

Runtime notes:

- pods carry `azure.workload.identity/use: "true"`
- the service account carries `azure.workload.identity/client-id`
- the Python backend uses `DefaultAzureCredential`
