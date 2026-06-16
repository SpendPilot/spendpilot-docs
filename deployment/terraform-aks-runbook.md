# Terraform AKS Runbook

Validated in Azure on June 8, 2026 against:

- subscription `e1f5b4be-e0ba-4ccb-8708-a949458fcd83`
- region `Central India` for AKS, PostgreSQL, Storage, Document Intelligence, and Front Door
- region `East US 2` for Azure AI Foundry model hosting

This is the exact deployment path that was used to bring the environment up successfully.

## What Terraform creates

- resource group, VNet, subnets, Log Analytics, ACR
- AKS with OIDC issuer and Workload Identity enabled
- user-assigned managed identity and federated credential
- PostgreSQL Flexible Server and database
- Blob Storage and document container
- Azure AI Foundry account and model deployment
- Azure AI Document Intelligence account
- Microsoft Entra app registrations and service principals
- kGateway on AKS
- application namespace, services, Gateway, HTTPRoutes, and workloads
- Azure Front Door Premium, WAF, origin group, routes, and origin

## Known live deployment choices

- AKS node pools use `Standard_D2s_v3`, not `Standard_D2s_v5`.
  The tested subscription had `0` vCPU quota for `Standard_DSv5` in `Central India`.
- PostgreSQL uses `GP_Standard_D2s_v3` with `ZoneRedundant` HA and geo-redundant backup enabled.
  The original `B_Standard_B2s` Burstable choice could not satisfy the approved HA target during a live apply.
- Azure AI Foundry stays in `East US 2`.
  The tested `gpt-4.1-mini` pay-per-token deployment succeeded there, while the Central India probe did not.
- kGateway is installed from the vendored chart under `repos/spendpilot-infra/vendor/kgateway/`.
  This avoids OCI chart pull failures in restricted networks.
- Front Door forwards to the origin with `HttpOnly`, and the origin group health probe also uses `Http`.
- The validated Front Door origin target is the gateway public IP's Azure cloudapp FQDN override, not the raw public IP.
- Terraform can bind validated apex and `www` Front Door custom-domain IDs into their route associations and into the shared WAF security policy.
- The gateway `LoadBalancer` service exposes `80` in the validated default path.
- The WAF policy includes a custom auth rate-limit rule on `/api/auth`.
- Blob Storage defaults to OAuth auth for the application path and keeps local users disabled.
  Shared-key auth remains enabled because the current AzureRM provider still uses it when managing queue properties.

## Prerequisites

1. Sign in to Azure CLI.
2. Select the correct subscription.
3. Fill `repos/spendpilot-infra/envs/dev/terraform.tfvars`.
4. Push your code branch to GitHub if you plan to use the ACR Task workaround.

Commands:

```powershell
az login
az account set --subscription <subscription-id>
cd repos/spendpilot-infra/envs/dev
terraform init
```

## Remote backend and workspaces

This stack now uses the Azure Blob remote backend:

- resource group: `terra-rg`
- storage account: `lijazterracount`
- container: `terracontainer`
- key: `spendpilot.tfstate`

Current workspace model:

- `dev` is the active live workspace and owns the existing deployed stack
- `default` is intentionally empty
- `staging` is an empty placeholder for future work
- `prod` is an empty placeholder for future work

Important:

- keep using `terraform workspace select dev` before planning or applying this live stack
- `staging` and `prod` are not ready to apply yet because the configuration is not yet parameterized by workspace-specific names or variables
- the current live infrastructure still uses the existing `spendpilot-prod-*` resource names even though it now lives under the `dev` Terraform workspace, because this migration was state-only and intentionally non-destructive

Helper script:

```powershell
cd repos/spendpilot-infra/envs/dev
powershell -ExecutionPolicy Bypass -File .\dev-workspace.ps1 -Action init
powershell -ExecutionPolicy Bypass -File .\dev-workspace.ps1 -Action plan
powershell -ExecutionPolicy Bypass -File .\dev-workspace.ps1 -Action apply
```

Script behavior:

- always runs `terraform init`
- always switches back to the live `dev` workspace
- always refreshes `.generated-kubeconfig` from the live AKS admin credentials before `plan` and `apply`
- uses Azure CLI auth for the backend during local runs
- switches the backend to OIDC plus Azure AD auth automatically when `GITHUB_ACTIONS=true`
- defaults to `build_images_during_apply=false` unless you pass `-BuildImagesDuringApply`

## GitHub Actions workflow

The repository now includes `.github/workflows/terraform-dev.yml`.

Behavior:

- `pull_request` into `main` runs `plan`
- only PR source branches starting with `terraform/` are allowed to run that plan job
- `push` to `main` runs `apply`
- `workflow_dispatch` can also run the same `apply` path manually
- the workflow uses Microsoft Entra OIDC through the `spendpilot-prod-github-actions` app registration and does not need a client secret
- the workflow runs `terraform init`, `terraform workspace select dev`, `az aks get-credentials`, and then `terraform plan` or `terraform apply` directly in YAML

Live OIDC values:

- client ID: `f2423c6d-2f93-4369-8ed5-b0637b086dcc`
- tenant ID: `920e9322-340c-4fbc-bf09-dc8fd6636182`
- subscription ID: `e1f5b4be-e0ba-4ccb-8708-a949458fcd83`

Federated credential subjects:

- `repo:SpendPilot/spend-control-platform:pull_request`
- `repo:SpendPilot/spend-control-platform:ref:refs/heads/main`

## Standard path

1. Run the first apply.

```powershell
terraform apply -var-file terraform.tfvars -auto-approve
```

2. If Helm or Kubernetes provider steps fail because the generated kubeconfig is missing, fetch admin credentials once and re-run apply.

```powershell
az aks get-credentials `
  --resource-group <resource-group> `
  --name <aks-name> `
  --admin `
  --file .generated-kubeconfig `
  --overwrite-existing

terraform apply -var-file terraform.tfvars -auto-approve
```

3. If local image builds fail because of SSL interception, skip local builds and use prebuilt images or ACR Tasks.

```powershell
terraform apply `
  -var-file terraform.tfvars `
  -var build_images_during_apply=false `
  -auto-approve
```

## ACR Task workaround for restricted laptops

Use this path when `az acr build` cannot reach the registry cleanly from the workstation.

1. Create the ACR tasks once.

```powershell
az acr task create `
  --registry <acr-name> `
  --name spend-control-backend-build `
  --context https://github.com/<org>/<repo>.git#<branch>:backend `
  --file Dockerfile `
  --image spend-control-backend:latest `
  --commit-trigger-enabled false `
  --pull-request-trigger-enabled false

az acr task create `
  --registry <acr-name> `
  --name spend-control-frontend-build `
  --context https://github.com/<org>/<repo>.git#<branch>:frontend `
  --file Dockerfile `
  --image spend-control-frontend:latest `
  --commit-trigger-enabled false `
  --pull-request-trigger-enabled false
```

2. Trigger both tasks.

```powershell
az acr task run --registry <acr-name> --name spend-control-backend-build
az acr task run --registry <acr-name> --name spend-control-frontend-build
```

3. Wait for both runs to succeed, then re-run Terraform with `build_images_during_apply=false`.

## Post-apply verification

Cluster checks:

```powershell
az aks command invoke -g <resource-group> -n <aks-name> --command "kubectl get pods -A"
az aks command invoke -g <resource-group> -n <aks-name> --command "kubectl get gateway,httproute,svc -n spend-control"
```

Direct gateway check:

```powershell
curl.exe -I http://<gateway-public-ip>
curl.exe http://<gateway-public-ip>/health
```

Application health checks used in the validated run:

```powershell
curl.exe https://myfinagent.online/health
curl.exe https://<frontdoor-default-domain>/runtime-config
```

Front Door check:

```powershell
curl.exe -I https://<frontdoor-default-domain>
```

PostgreSQL posture check:

```powershell
az postgres flexible-server show `
  -g <resource-group> `
  -n <postgres-server-name> `
  --query "{sku:sku.name,tier:sku.tier,ha:highAvailability.mode,geoBackup:backup.geoRedundantBackup,state:state}"
```

## Manual work after Terraform

Terraform does not manage these external steps:

- map `myfinagent.online` to Azure Front Door from Hostinger
- add both `myfinagent.online` and `www.myfinagent.online` as Front Door custom domains
- validate TLS for both domains
- copy the resulting custom-domain resource IDs into `frontdoor_apex_custom_domain_id` and `frontdoor_www_custom_domain_id` in `terraform.tfvars`
- assign customer users in their own Entra tenants after they consent to the app
- if a customer tenant blocks user consent or has not onboarded the app yet, have a tenant admin open the `entra_admin_consent_url_template` output with their tenant ID filled in so Entra can create the external enterprise applications and service principals

## Important change note

The June 8, 2026 hardening pass uplifted PostgreSQL from the original Burstable SKU to `GP_Standard_D2s_v3` so HA plus geo-backup could succeed. In the validated environment that required destroying and recreating the PostgreSQL server and database, which reset application data in that environment.

## Front Door note

After a successful apply, Front Door configuration can still take time to propagate globally. During validation on June 8, 2026:

- the gateway public IP returned `200 OK` immediately
- the Front Door default hostname could still return a Front Door `404` during early propagation

Treat the direct gateway health check as the immediate truth, and re-test Front Door after propagation completes.

If the Front Door default hostname or one of the custom domains still returns the Azure-managed `404 CONFIG_NOCACHE` page after roughly 45 minutes:

- re-save the route and origin in the portal or by Terraform apply
- verify the custom domain is attached to the active route, not just validated on the profile
- verify the gateway public IP still answers `200 OK`
- open an Azure support case for Front Door route propagation with the endpoint, route, and origin IDs
