# Deployment context

Canonical production path:

- `repos/spendpilot-infra/envs/dev/` is the current active Azure bootstrap root and still runs Helm as part of the live path
- `repos/spendpilot-helm/charts/spendpilot/` defines the in-cluster application topology
- `repos/spendpilot-infra/k8s/` mirrors the same shape as raw YAML

Key deployment choices:

- no Kubernetes Ingress
- Gateway API and kGateway only
- Front Door Premium with WAF as the public edge
- Azure Blob remote backend for Terraform state
- Front Door terminates browser HTTPS at the edge and forwards to the gateway over HTTP in the validated path
- backend is split into `identity`, `finance`, and `documents` services on AKS
- each backend service now builds from its own folder and Dockerfile under `repos/spendpilot-services/services/`
- local docker compose stays combined for simplicity
- kGateway charts are vendored from the official `kgateway-dev/kgateway` `v2.3.0` source tree under `repos/spendpilot-infra/vendor/kgateway/`
- the app namespace is created by Terraform with `kubernetes_namespace`; the Helm release does not create it
- the Helm migration job must stay `post-install,post-upgrade`

Live-tested Azure findings from June 8, 2026:

- `Central India` in the tested subscription had no usable `Standard_DSv5` quota, so AKS uses `Standard_D2s_v3`
- PostgreSQL HA plus geo-backup required a move from the original Burstable SKU to `GP_Standard_D2s_v3`
- the tested Azure AI Foundry deployment path for `gpt-4.1-mini` worked in `East US 2`, not in the attempted Central India configuration
- local `az acr build` can fail behind SSL interception; GitHub-backed ACR Tasks are the reliable fallback
- Front Door can return a temporary platform `404` while edge configuration propagates even after the Azure resource `provisioningState` is `Succeeded`
- the validated stable Front Door origin target is the gateway public IP's Azure cloudapp FQDN, not the raw public IP
- the validated live custom-domain routing keeps apex on the primary Front Door route and `www` on its own dedicated route, and Terraform can now bind those custom-domain IDs into both the route layer and the WAF security policy
- the frontend must not rely on build-time `NEXT_PUBLIC_*` injection for Entra config; runtime config is served from `/runtime-config`
- Terraform now uses frontend and backend source-tree hashes both to trigger image rebuilds and to force Helm rollouts when source changes
- the gateway TLS secret can still be created during Terraform apply through Azure CLI plus `kubectl apply`, but it is not required in the default validated Front Door path
- the storage account defaults to OAuth auth and disables local users, but shared-key auth stays enabled because the current AzureRM provider still expects it for storage-account management
- the Terraform backend now lives in Azure Blob Storage under `terraform-rg` / `lijaztf` / `states` / `spendpilot.tfstate`
- `dev` is the only live Terraform workspace; `default` is intentionally empty and `staging` / `prod` are empty placeholders for future workspace-aware expansion
- `repos/spendpilot-infra/envs/dev/dev-workspace.ps1` is still the current operator helper, but it is now review-classified because it preserves the old workspace-based flow inside the new env-root layout
- `.github/workflows/terraform-dev.yml` is the canonical GitHub Actions path for the live environment
- Azure auth for that workflow uses Microsoft Entra OIDC via the `spendpilot-prod-github-actions` app registration
- the GitHub OIDC subjects currently trusted are `repo:SpendPilot/spend-control-platform:pull_request` and `repo:SpendPilot/spend-control-platform:ref:refs/heads/main`
