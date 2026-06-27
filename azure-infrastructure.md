# Azure Infrastructure

Canonical Azure target:

```txt
Azure Front Door Premium + WAF
  -> HTTP to the kGateway public service on AKS through the gateway Azure cloudapp FQDN
  -> HTTPRoutes
      -> frontend
      -> identity-service
      -> finance-service
      -> documents-service

AKS
  -> private control plane with API server VNet integration in prod
  -> user-assigned managed identity via Workload Identity
  -> Azure managed Prometheus + Azure Managed Grafana in prod
  -> PostgreSQL Flexible Server (General Purpose, zone-redundant HA, geo-backup)
  -> Blob Storage
  -> Azure AI Foundry account + model deployment
  -> Azure AI Document Intelligence account
```

Provisioning source of truth:

- `repos/spendpilot-infra/envs/dev/`
- `repos/spendpilot-infra/envs/prod/`

Application deployment source of truth:

- `repos/spendpilot-helm/charts/spendpilot/`

Key design points:

- minimal AKS node pools: one system pool and one user pool
- `Standard_D2s_v3` node sizing in the validated subscription because `Standard_DSv5` quota was unavailable in `Central India`
- PostgreSQL uses `GP_Standard_D2s_v3` because the original Burstable SKU could not satisfy the approved HA + geo-backup target
- Central India remains the default region for the core platform
- Azure AI Foundry is intentionally separated to `East US 2` in the validated path
- one managed identity shared by the application pods
- Entra multi-tenant app registrations created during Terraform apply
- Terraform uses Azure CLI during apply for `az acr build` and `az aks command invoke`
- the validated fallback for restricted laptops is GitHub-backed ACR Tasks plus `build_images_during_apply=false`
- kGateway is installed from the vendored official chart source in `repos/spendpilot-infra/vendor/kgateway/`
- Front Door probes the AKS origin over `HTTP` on `/health`
- the validated stable origin target is the gateway public IP's Azure cloudapp FQDN, not the raw public IP
- the WAF policy includes an auth rate-limit rule for `/api/auth`
- the current live production Front Door custom domain is `costpilot.online`
- the current live production AKS API endpoint is private and resolves through the AKS-managed private DNS zone
- prod operator access to the private cluster is through the Australia ops jumpbox VNet plus Azure control-plane commands, not a public Kubernetes API endpoint
- the storage account defaults to OAuth auth and keeps local users disabled; shared-key auth stays enabled only to preserve Terraform/AzureRM management compatibility
- the current live production Key Vault is `spendpilot-prod-kv-2300`
- the current live production PostgreSQL server is `spendpilot-prod-pgsql-2300` on `Standard_D2s_v3` with geo-redundant backup enabled and high availability currently disabled
- the current live production monitoring stack is Azure Monitor workspace `spendpilot-prod-amw` plus Azure Managed Grafana `spendpilot-prod-grafana`
- `envs/staging` is presently a placeholder Terraform root with no live Azure resource group behind it
