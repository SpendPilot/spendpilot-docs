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
  -> user-assigned managed identity via Workload Identity
  -> PostgreSQL Flexible Server (General Purpose, zone-redundant HA, geo-backup)
  -> Blob Storage
  -> Azure AI Foundry account + model deployment
  -> Azure AI Document Intelligence account
```

Provisioning source of truth:

- `repos/spendpilot-infra/envs/dev/`

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
- validated custom-domain wiring keeps `myfinagent.online` on the primary Front Door route and `www.myfinagent.online` on a dedicated Front Door route; both domains are associated with the same WAF security policy
- the storage account defaults to OAuth auth and keeps local users disabled; shared-key auth stays enabled only to preserve Terraform/AzureRM management compatibility
