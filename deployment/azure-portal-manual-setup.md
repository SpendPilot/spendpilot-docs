# Azure Portal Manual Setup Guide

This guide is the portal-first fallback when you do not want to rely on Terraform for the Azure control plane.

It was updated after a live deployment validation on June 8, 2026 and reflects the working topology in this repository.

## Target architecture

```txt
Azure Front Door Premium + WAF
  -> AKS public LoadBalancer service for kGateway
  -> Gateway API HTTPRoutes
      -> frontend
      -> identity-service
      -> finance-service
      -> documents-service
```

## Portal resources to create

Create these in order:

1. Resource group in `Central India`
2. Log Analytics workspace
3. Container Registry
4. Virtual network with:
   - AKS subnet
   - PostgreSQL subnet
5. User-assigned managed identity
6. PostgreSQL Flexible Server in the delegated subnet
7. Storage account and blob container
8. Azure AI Document Intelligence in `Central India`
9. Azure AI Foundry account in `East US 2`
10. AKS cluster with Workload Identity and OIDC issuer enabled
11. Azure Front Door Premium profile, endpoint, WAF policy, origin group, origin, route
12. Microsoft Entra app registrations for frontend and backend APIs

## Critical portal settings

AKS:

- Kubernetes version: use the latest supported stable version that is compatible with your chosen kGateway release
- OIDC issuer: `Enabled`
- Workload Identity: `Enabled`
- Node pools: one `system` pool and one `user` pool
- Tested working VM size in this subscription: `Standard_D2s_v3`

Managed identity:

- create one user-assigned identity for the workloads
- grant it:
  - `Storage Blob Data Contributor` on the storage account
  - `Cognitive Services OpenAI User` on the Foundry account
  - `Cognitive Services User` on the Document Intelligence account

PostgreSQL:

- use private access in the delegated subnet
- keep the app connection on SSL
- use a General Purpose SKU that supports HA in your region
- validated working shape in this repo:
  - SKU: `Standard_D2s_v3` under the General Purpose tier
  - primary zone: `1`
  - standby zone: `2`
  - HA: `ZoneRedundant`
  - geo-redundant backup: `Enabled`

Storage:

- set default authentication to OAuth
- disable local users
- if you need the same Terraform/AzureRM behavior used in this repo, keep shared-key auth enabled even though the application itself uses managed identity

Foundry:

- use `East US 2` if you want the same tested path as this repo
- deploy `gpt-4.1-mini`

Front Door:

- SKU: Premium
- WAF: attach to the endpoint
- origin protocol: HTTP to the AKS gateway LoadBalancer
- health probe protocol: HTTP
- health probe path: `/health`
- route pattern: `/*`
- add a custom WAF rate-limit rule for `/api/auth`
- prefer an Azure DNS name on the gateway public IP, then use that FQDN as the Front Door origin host and origin host header

Gateway:

- expose port `80` on the public `LoadBalancer` service for the validated default path
- only add a TLS listener on `443` if you also supply a publicly trusted origin certificate

## Manual Entra setup

Create two app registrations:

1. Frontend SPA app
   - supported account types: Accounts in any organizational directory and personal Microsoft accounts
   - set access token version to `2`
   - platform: Single-page application
   - redirect URIs:
     - `https://<frontdoor-domain>/login`
     - local development URI if needed

2. Backend API app
   - supported account types: Accounts in any organizational directory and personal Microsoft accounts
   - expose an API application ID URI
   - set access token version to `2`
   - add the frontend SPA app ID to known client applications
   - pre-authorize the frontend SPA app for the backend `access_as_user` delegated scope
   - create app roles if you want tenant-side role assignment

Grant admin consent after both apps are configured in your home tenant.

Important:

- use the `common` authority in the frontend configuration
- set the frontend API scope to `<backend Application ID URI>/access_as_user`
- customer-company users still group by their Entra tenant
- direct personal Microsoft-account sign-ins get one isolated workspace each
- guest Microsoft accounts invited into a customer Entra tenant stay in that tenant workspace
- end users do not need to be pre-registered in the app database
- first successful browser sign-in bootstraps the app-side account automatically
- customer tenants may still require tenant-admin consent if their Entra policies block user consent
- cross-tenant service principals are created by consent inside the customer tenant; you cannot pre-create them from your own tenant without their admin participating

## Images and registry

If the workstation cannot push images directly because of SSL inspection or Docker restrictions, use ACR Tasks from the portal or Azure CLI.

Recommended path:

1. Push the repo branch to GitHub.
2. In ACR, create one task for `repos/spendpilot-frontend/` and three service-image tasks for `repos/spendpilot-services/`.
3. Build `spend-control-identity:latest`, `spend-control-finance:latest`, `spend-control-documents:latest`, and `spend-control-frontend:latest`.

## In-cluster deployment

The Azure portal does not replace these Kubernetes steps. Use Cloud Shell or a trusted machine with `kubectl` and `helm`.

1. Get AKS admin credentials.
2. Install Gateway API CRDs.
3. Install the vendored kGateway chart from `repos/spendpilot-infra/vendor/kgateway/`.
4. Deploy the app Helm chart from `repos/spendpilot-helm/charts/spendpilot/`.

Important:

- the application namespace must exist before the app Helm release runs
- the app chart should not also ask Helm to create the namespace
- the migration job must run as a post-install or post-upgrade hook

## Hostinger DNS work after Azure

Terraform and the portal do not control Hostinger. After Front Door is up:

1. Add `myfinagent.online` as a custom domain on Front Door.
2. Add `www.myfinagent.online` as a second custom domain on Front Door.
3. Create the required DNS validation record in Hostinger for each custom domain.
4. Point both hostnames to the Front Door hostname.
5. Record the Azure resource IDs of both Front Door custom domains so Terraform can bind them to the right routes and WAF association later.
6. Attach `myfinagent.online` to the primary Front Door route.
7. If `www.myfinagent.online` still serves the Azure `404 CONFIG_NOCACHE` page on the shared route, create a dedicated `www` Front Door route and attach only `www.myfinagent.online` to it.
8. Wait for certificate issuance and route propagation.

## Trustworthy manual checks

Use these checks after the portal build:

- AKS pods are all `Running`
- `kubectl get gateway,httproute,svc -n spend-control` shows the gateway programmed
- the gateway public IP returns `200 OK`
- the Front Door hostname returns the frontend after propagation
- `https://myfinagent.online/health` returns the identity health payload
- a first-time user can finish the browser consent flow and then `GET /api/auth/me` returns JSON instead of `401 Invalid access token`
- if a customer tenant sees `AADSTS650052`, a tenant admin must use the admin-consent URL for that tenant so Entra creates the frontend and backend enterprise applications there

If Front Door keeps returning the Azure error page with `X-Cache: CONFIG_NOCACHE` for more than roughly 45 minutes while the gateway public IP is healthy, treat it as an Azure Front Door propagation issue and escalate through Azure support.
