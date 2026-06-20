# SpendPilot Platform Routes And Environments

Last updated: 2026-06-20

## Scope

This runbook captures the intended routing, environment, GitOps, and rollout model for the split SpendPilot workspace under `SpendMicro`.

Repos in scope:

- `spendpilot-frontend/`
- `spendpilot-services/`
- `spendpilot-helm/`
- `spendpilot-infra/`
- `spendpilot-gitops/`
- `spendpilot-docs/`

## Environment Model

- shared global foundation: `spendpilot-infra/envs/global-shared/`
- shared nonprod edge: `spendpilot-infra/envs/nonprod-shared/`
- dev runtime: `spendpilot-infra/envs/dev/`
- staging runtime: `spendpilot-infra/envs/staging/`
- prod runtime: `spendpilot-infra/envs/prod/`

Nonprod means the `dev` and `staging` runtime roots plus the shared nonprod routing layer in `nonprod-shared`.

## Hostnames

- prod: `costpilot.online`
- dev: `dev.costpilot.online`
- staging: `stage.costpilot.online`

Current rollback/fallback hostname that still matters operationally:

- prod App Gateway fallback: `fin.nexaflow.site`

## Edge And Routing Model

### Production

Target public edge:

- Azure Front Door

Target route flow:

- `costpilot.online`
- Azure Front Door custom domain
- Azure Front Door route
- Azure Front Door origin group
- prod kGateway public LoadBalancer IP
- in-cluster `Gateway` and `HTTPRoute`
- frontend and backend services

Current protected fallback:

- Azure Application Gateway remains in place as the working rollback edge
- the checked-in backend target contract is the prod kGateway public IP `4.247.241.143`

### Nonprod

Target shared public edge:

- Azure Front Door in `spendpilot-infra/envs/nonprod-shared/`

Target route flow:

- `dev.costpilot.online`
- nonprod Front Door dev custom domain
- nonprod Front Door dev route
- nonprod Front Door dev origin group
- dev kGateway public endpoint

and

- `stage.costpilot.online`
- nonprod Front Door staging custom domain
- nonprod Front Door staging route
- nonprod Front Door staging origin group
- staging kGateway public endpoint

Safety rule:

- dev and staging origins must never point to prod

## kGateway Dependency

- SpendPilot uses kGateway for in-cluster entry and routing.
- Azure Front Door and the prod fallback Application Gateway both depend on the environment kGateway public endpoint contract.
- Helm now renders HTTPRoute hostnames from `gateway.hostnames` so GitOps environment values control hostname matching.

## TLS And Custom Domain Requirements

### Front Door

- Azure Front Door custom domains require domain validation.
- Validation typically needs Azure-issued TXT records.
- Managed certificate issuance depends on successful domain validation.
- Apex handling for `costpilot.online` may require Azure DNS alias support or registrar flattening support depending on where the zone is hosted.

### Application Gateway Fallback

- Prod fallback TLS is still represented through the existing Application Gateway and Key Vault certificate flow for `fin.nexaflow.site`.

## Expected DNS Records

Verify with the actual DNS hosting provider before apply or cutover.

Likely records:

- `costpilot.online`
  - Azure Front Door validation TXT record
  - Azure Front Door route record or alias/CNAME-equivalent depending on provider support for apex
- `dev.costpilot.online`
  - Azure Front Door validation TXT record
  - CNAME to the nonprod Front Door endpoint host
- `stage.costpilot.online`
  - Azure Front Door validation TXT record
  - CNAME to the nonprod Front Door endpoint host

## Apply Order

### Dev-first rollout

1. `global-shared`
2. `nonprod-shared`
3. `dev`
4. GitOps merge for dev image values
5. Argo CD dev auto-sync validation

### Staging rollout

1. confirm staging runtime endpoint contract is ready
2. `staging`
3. `nonprod-shared` if the staging route toggle or origin contract changed
4. GitOps staging promotion PR merge
5. Argo CD staging auto-sync validation

### Prod rollout

1. confirm Azure Front Door quota is available
2. confirm `costpilot.online` validation records can be created
3. `prod`
4. validate Front Door custom domain and managed certificate readiness
5. update DNS cutover for `costpilot.online`
6. validate production through Front Door
7. keep Application Gateway fallback in place until cutover confidence is high

## Terraform Plan And Apply Commands

Run from the workspace root:

```powershell
terraform -chdir=spendpilot-infra/envs/global-shared fmt
terraform -chdir=spendpilot-infra/envs/global-shared validate
terraform -chdir=spendpilot-infra/envs/global-shared plan -out global-shared.plan
terraform -chdir=spendpilot-infra/envs/global-shared apply global-shared.plan
```

```powershell
terraform -chdir=spendpilot-infra/envs/nonprod-shared fmt
terraform -chdir=spendpilot-infra/envs/nonprod-shared validate
terraform -chdir=spendpilot-infra/envs/nonprod-shared plan -out nonprod-shared.plan
terraform -chdir=spendpilot-infra/envs/nonprod-shared apply nonprod-shared.plan
```

```powershell
terraform -chdir=spendpilot-infra/envs/dev fmt
terraform -chdir=spendpilot-infra/envs/dev validate
terraform -chdir=spendpilot-infra/envs/dev plan -out dev.plan
terraform -chdir=spendpilot-infra/envs/dev apply dev.plan
```

```powershell
terraform -chdir=spendpilot-infra/envs/staging fmt
terraform -chdir=spendpilot-infra/envs/staging validate
terraform -chdir=spendpilot-infra/envs/staging plan -out staging.plan
terraform -chdir=spendpilot-infra/envs/staging apply staging.plan
```

```powershell
terraform -chdir=spendpilot-infra/envs/prod fmt
terraform -chdir=spendpilot-infra/envs/prod validate
terraform -chdir=spendpilot-infra/envs/prod plan -out prod.plan
terraform -chdir=spendpilot-infra/envs/prod apply prod.plan
```

Local workstation note:

- `spendpilot-infra/envs/dev validate` was blocked during this change set by a local Terraform provider checksum/cache mismatch.
- fix the local provider cache issue before treating a local dev validate failure as a code defect

## Helm And GitOps Validation Commands

```powershell
helm template spendpilot-dev spendpilot-helm/charts/spendpilot -f spendpilot-gitops/environments/dev/values/spendpilot-values.yaml > $null
helm template spendpilot-staging spendpilot-helm/charts/spendpilot -f spendpilot-gitops/environments/staging/values/spendpilot-values.yaml > $null
helm template spendpilot-prod spendpilot-helm/charts/spendpilot -f spendpilot-gitops/environments/prod/values/spendpilot-values.yaml > $null
```

```powershell
rg -n "automated|prune|selfHeal|CreateNamespace" spendpilot-gitops/environments/dev/applications/spendpilot.yaml spendpilot-gitops/environments/staging/applications/spendpilot.yaml spendpilot-gitops/environments/prod/applications/spendpilot.yaml
```

```powershell
rg -n "hostnames:|backendCorsOrigins|replicaCount:|resources:" spendpilot-gitops/environments/dev/values/spendpilot-values.yaml spendpilot-gitops/environments/staging/values/spendpilot-values.yaml spendpilot-gitops/environments/prod/values/spendpilot-values.yaml
```

Current workstation note:

- `helm` was not installed locally during this change set, so the Helm render commands were documented but not executed here

## Argo CD Sync Model

- dev app manifest: auto-sync enabled
- staging app manifest: auto-sync enabled
- prod app manifest: auto-sync enabled

Deployment behavior:

- GitHub Actions update GitOps state
- Argo CD reconciles merged GitOps changes
- no GitHub Actions workflow should deploy directly to Kubernetes

## Capacity Model

### Prod

- AKS sizing remains unchanged unless a defect requires adjustment
- checked-in application target:
  - frontend: minimum `2` replicas with HPA enabled
  - identity: `2` replicas
  - finance: `2` replicas
  - documents: `2` replicas
- checked-in resource envelope per deployment:
  - CPU requests `>=100m`
  - CPU limits `<=500m`
  - memory requests `>=128Mi`
  - memory limits `<=512Mi`

### Dev

- AKS system pool: min `1`, max `1`
- AKS user pool: min `0`, max `1`
- checked-in application target:
  - frontend: minimum `2` replicas with HPA enabled
  - identity: `2` replicas
  - finance: `2` replicas
  - documents: `2` replicas
- GitOps values now use the same bounded resource envelope:
  - CPU requests `>=100m`
  - CPU limits `<=500m`
  - memory requests `>=128Mi`
  - memory limits `<=512Mi`

### Staging

- lower than prod
- app workloads: one replica each in the current checked-in GitOps values
- GitOps values include slightly roomier limits than dev while staying below prod intent

## Rollback Approach

### Dev And Staging

- revert the GitOps values PR or commit
- if needed, revert the related Helm or Terraform change in git
- re-run the relevant Terraform plan before apply

### Production

- keep the existing Application Gateway path available during Front Door rollout
- if Front Door validation or cutover fails:
  - leave DNS on the fallback path
  - revert the Front Door-related Terraform change in git if needed
  - preserve the current Application Gateway listener, TLS, and backend configuration until rollback is complete

## Post-Deploy Validation

### Dev

- confirm the dev Front Door route resolves to `dev.costpilot.online`
- confirm the kGateway route matches the hostname
- confirm `/runtime-config` responds with dev-safe host data
- confirm Argo CD reports the dev app `Synced` and `Healthy`

### Staging

- confirm the staging Front Door route resolves to `stage.costpilot.online`
- confirm the kGateway route matches the hostname
- confirm staging uses the promoted dev digest
- confirm Argo CD reports the staging app `Synced` and `Healthy`

### Prod

- confirm Azure Front Door custom domain is validated
- confirm the managed certificate is issued
- confirm `costpilot.online` resolves to the intended Front Door endpoint
- confirm requests route to the prod kGateway endpoint
- confirm `/runtime-config` serves the expected prod auth and API contract
- confirm the Application Gateway fallback still works until formally retired

## Database DR Notes

- dev and prod now have a checked-in Terraform DR design for PostgreSQL read replicas in `South India`
- this DR path was validated syntactically with Terraform, but not applied from this change set
- after apply, validate:
  - replica server exists
  - replica FQDN is output
  - replication lag/health is acceptable in Azure
  - the replica remains read-only until an explicit disaster recovery promotion action
