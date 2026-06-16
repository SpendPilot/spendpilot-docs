PROJECT_NAME: SpendPilot

You are a senior Azure cloud infrastructure engineer, Terraform engineer, Kubernetes engineer, Helm engineer, GitOps engineer, CI/CD engineer, and DevOps architect.

Before making changes, read these context files:

* AI_CONTEXT.md
* CURRENT_STATE.md
* TARGET_ARCHITECTURE.md
* DECISIONS.md
* REFACTOR_PLAN.md
* RISK_REGISTER.md
* MIGRATION_CHECKLIST.md
* CLEANUP_STRATEGY.md
* SERVICE_BOUNDARIES.md
* SERVICE_SPLIT_READINESS.md
* REPO_SPLIT_PLAN.md
* CLEANUP_PLAN.md
* CLEANUP_REPORT.md if it exists

Goal:
Refactor infrastructure, Helm, GitOps, and CI/CD into a clean Azure production-minded structure. Remove old redundant deployment files after validation so only required files remain.

Fixed platform architecture:

* Azure
* Terraform
* Azure Blob Storage remote backend for Terraform state
* No Terraform workspaces for environment separation
* AKS
* ACR
* kGateway
* ArgoCD
* GitHub Actions
* GitHub Actions authentication to Azure through OIDC where possible
* dev, staging, and prod environments
* ACR shared globally
* non-prod Front Door shared by dev and staging
* non-prod AI services shared by dev and staging
* prod Front Door inside envs/prod
* prod AKS public for now
* Terraform bootstraps platform components
* ArgoCD deploys application workloads

Fixed principles:

1. Infrastructure code must be separated from application code.
2. Terraform must not deploy application microservices directly.
3. Terraform should provision Azure resources and platform bootstrap components.
4. Terraform may bootstrap AKS, kGateway, and ArgoCD.
5. Application deployment should happen through Helm/GitOps.
6. Environments must have separate root modules/state files instead of Terraform workspaces.
7. Shared resources must have their own root module/state.
8. Production should be secure but not overengineered.
9. Naming should be simple and based on PROJECT_NAME.
10. Old infrastructure, Helm, Kubernetes, and CI/CD files must be cleaned up after validation.
11. Do not perform dangerous state operations without documenting them first.
12. Do not allow two Terraform states to manage the same Azure resource.

Target infrastructure repo layout:

repos/<PROJECT_NAME>-infra/
modules/
resource-group/
networking/
aks/
acr/
postgres/
storage/
keyvault/
ai-services/
monitoring/
managed-identity/
workload-identity/
frontdoor/
private-endpoint/
private-dns/
kgateway-bootstrap/
argocd-bootstrap/

envs/
global-shared/
nonprod-shared/
dev/
staging/
prod/

Environment responsibility model:

1. global-shared

Owns resources shared by all environments:

* ACR
* global/common managed identities if needed
* shared DNS only if required
* shared state references/outputs
* other truly global resources

2. nonprod-shared

Owns resources shared only by dev and staging:

* non-prod Azure Front Door
* non-prod WAF policy
* shared non-prod Azure AI Document Intelligence if used
* shared non-prod Azure AI Foundry/OpenAI services if used
* shared non-prod monitoring only if cost-saving is required and acceptable
* other cost-saving shared non-prod services

3. dev

Owns dev runtime:

* resource group
* VNet/subnets
* AKS public cluster
* PostgreSQL/database resources if used
* storage/blob resources if used
* Key Vault if used
* managed identities
* workload identity setup
* monitoring
* kGateway bootstrap
* ArgoCD bootstrap
* outputs needed by shared non-prod Front Door

4. staging

Owns staging runtime:

* resource group
* VNet/subnets
* AKS public cluster
* PostgreSQL/database resources if used
* storage/blob resources if used
* Key Vault if used
* managed identities
* workload identity setup
* monitoring
* kGateway bootstrap
* ArgoCD bootstrap
* outputs needed by shared non-prod Front Door

5. prod

Owns prod runtime and prod edge:

* resource group
* VNet/subnets
* AKS public cluster for now
* PostgreSQL/database resources if used
* storage/blob resources if used
* Key Vault if used
* managed identities
* workload identity setup
* monitoring
* kGateway bootstrap
* ArgoCD bootstrap
* Azure Front Door prod
* WAF prod
* prod routes/origins
* private endpoints/private DNS where useful and not overengineered
* production security controls

Naming guidance:
Use simple names based on PROJECT_NAME.

Examples:

* rg-<PROJECT_NAME>-global
* rg-<PROJECT_NAME>-nonprod-shared
* rg-<PROJECT_NAME>-dev
* rg-<PROJECT_NAME>-staging
* rg-<PROJECT_NAME>-prod
* acr<PROJECT_NAME><suffix>
* aks-<PROJECT_NAME>-dev
* aks-<PROJECT_NAME>-staging
* aks-<PROJECT_NAME>-prod
* vnet-<PROJECT_NAME>-dev
* vnet-<PROJECT_NAME>-staging
* vnet-<PROJECT_NAME>-prod
* afd-<PROJECT_NAME>-nonprod
* afd-<PROJECT_NAME>-prod
* kv-<PROJECT_NAME>-dev-<suffix>
* kv-<PROJECT_NAME>-staging-<suffix>
* kv-<PROJECT_NAME>-prod-<suffix>
* pg-<PROJECT_NAME>-dev
* pg-<PROJECT_NAME>-staging
* pg-<PROJECT_NAME>-prod
* log-<PROJECT_NAME>-dev
* log-<PROJECT_NAME>-staging
* log-<PROJECT_NAME>-prod

Convert PROJECT_NAME to lowercase/alphanumeric where Azure naming requires it.
Use short random suffixes only for globally unique Azure resources.

Shared Front Door / origin dependency rule:

* nonprod-shared owns the whole non-prod Front Door.
* dev and staging must not directly mutate the shared Front Door.
* dev and staging must output their origin/backend details.
* nonprod-shared should read dev/staging outputs through terraform_remote_state or another controlled dependency mechanism.
* nonprod-shared then creates/updates origin groups, origins, routes, and WAF/security associations.
* prod owns its own Front Door inside envs/prod.
* Avoid circular dependencies.
* Avoid multiple Terraform states managing the same Front Door resource.

State strategy:

* Remove workspace-based environment management.
* Use separate env folders.
* Use separate backend state keys.

State keys:

* global-shared.tfstate
* nonprod-shared.tfstate
* dev.tfstate
* staging.tfstate
* prod.tfstate

Apply order:

1. global-shared
2. nonprod-shared initial apply
3. dev
4. staging
5. nonprod-shared second apply to attach dev/staging origins
6. prod

Production-first setup order:

1. global-shared
2. prod

For public prod AKS:

* Keep API server public for now.
* Add authorized IP ranges if possible.
* Enable Azure RBAC where appropriate.
* Disable local admin accounts if feasible.
* Use managed identity.
* Use workload identity.
* Avoid long-lived Azure secrets inside Kubernetes.
* Document future migration to private AKS.

Create/update these documents:

1. INFRA_REFACTOR_PLAN.md

Include:

* Current infra layout.
* Target infra layout.
* Module strategy.
* Environment strategy.
* State strategy.
* Workspace removal plan if applicable.
* Shared-resource strategy.
* Apply order.
* Validation commands.
* Risks.
* Rollback notes.

2. STATE_MIGRATION_PLAN.md

Include:

* Current state situation.
* Current workspace usage if present.
* Target state files.
* Backend configuration strategy.
* State backup steps.
* Migration steps.
* terraform state mv/import guidance if applicable.
* Rollback notes.
* Manual review warnings.
* Clear warning that destructive state commands must not run automatically without explicit approval.

3. SHARED_RESOURCE_STRATEGY.md

Include:

* Global shared resources.
* Non-prod shared resources.
* Environment-specific resources.
* How dependencies are passed through outputs.
* How non-prod shared Front Door reads dev/staging origins.
* How prod owns its own Front Door.
* How to avoid circular dependencies.
* How to avoid two states managing the same Azure resource.
* Apply order.

4. FRONTDOOR_ORIGIN_STRATEGY.md

Include:

* Non-prod Front Door ownership.
* Dev/staging origin output contract.
* Example output names.
* Example remote state read pattern.
* Origin group strategy.
* Route strategy.
* WAF/security policy strategy.
* Prod Front Door inside envs/prod.
* Validation checklist.

5. SECURITY_BASELINE.md

Include:

* Dev security baseline.
* Staging security baseline.
* Prod security baseline.
* Public prod AKS controls.
* Private endpoint/private DNS strategy where useful.
* Key Vault/secret handling.
* Managed identity.
* Workload identity.
* CI/CD secret handling.
* GitHub environment approvals.
* Future hardening TODOs.

6. HELM_REFACTOR_PLAN.md

Target:

repos/<PROJECT_NAME>-helm/
charts/
<PROJECT_NAME>/
Chart.yaml
values.yaml
values-dev.yaml
values-staging.yaml
values-prod.yaml
templates/
frontend/
services/
gateway/
httproutes/
config/
secrets/

Prefer one umbrella Helm chart first unless the project clearly requires separate charts.

Include:

* Current Helm/Kubernetes layout.
* Target chart layout.
* Values strategy.
* Environment override strategy.
* Image tag strategy.
* Secrets/config strategy.
* kGateway/Gateway API/HTTPRoute strategy.
* Validation commands.

7. GITOPS_STRATEGY.md

Target:

repos/<PROJECT_NAME>-gitops/
environments/
dev/
applications/
values/
staging/
applications/
values/
prod/
applications/
values/

Include:

* ArgoCD setup.
* One ArgoCD per AKS cluster unless the project clearly needs central ArgoCD.
* App-of-apps or Application layout.
* Dev auto-sync strategy if appropriate.
* Staging manual or controlled sync strategy.
* Prod manual sync/approval strategy.
* Image tag update flow.
* Promotion flow from dev to staging to prod.
* Rollback flow.

8. CICD_STRATEGY.md

Include:

* Frontend CI.
* Backend/services CI.
* Path-filtered builds for grouped services.
* Docker image build and push to ACR.
* Immutable image tags.
* No latest tag for production.
* Helm lint/template.
* Infrastructure plan/apply.
* GitOps validation.
* Environment approvals.
* Azure authentication through GitHub OIDC.
* Secret handling.
* Pull request checks.
* Main branch merge flow.
* Promotion flow.
* Rollback flow.

9. CLEANUP_INFRA_DEPLOYMENT_PLAN.md

Build a cleanup table for infra/deployment files:

* Path
* Classification: KEEP / MOVE / MERGE / DELETE / REVIEW
* Reason
* Replacement path
* References found
* Safe to remove now: yes/no
* Validation required

Cleanup targets:

* old Terraform workspace files
* old duplicated root modules
* unused Terraform variables/outputs
* old provider/backend files
* unused tfvars
* obsolete Kubernetes manifests replaced by Helm/GitOps
* old Helm charts replaced by new chart
* old Helm values files
* old CI/CD workflow files
* old deployment scripts
* old Docker build scripts
* stale environment files
* duplicate docs
* generated build artifacts
* cache folders
* empty folders
* old ArgoCD manifests replaced by new structure
* old ingress manifests replaced by Gateway API/HTTPRoute
* old direct Helm deployment scripts replaced by GitOps

Rules before deleting:

1. Search references across the repository.
2. Confirm replacements exist.
3. Run validation where possible.
4. Do not delete Terraform state files automatically.
5. Do not delete secrets automatically.
6. Do not delete unknown production files automatically.
7. Put uncertain files in REVIEW.
8. The final repo/folder must not contain duplicate active deployment systems unless intentionally documented.
9. Do not leave old workflows that can accidentally deploy.
10. Do not leave old Terraform roots that can accidentally manage the same infrastructure.

Implementation tasks:

1. Create target infra folder layout.
2. Move existing working infra into the correct env folder first, usually dev if that is the currently working environment.
3. Fix relative paths.
4. Extract modules gradually.
5. Create global-shared root.
6. Create nonprod-shared root.
7. Create dev root.
8. Create staging root.
9. Create prod root.
10. Add outputs for environment origin/backend details.
11. Add remote-state or controlled dependency reads for shared resources.
12. Add or refactor ACR in global-shared.
13. Add or refactor non-prod shared Front Door in nonprod-shared.
14. Add or refactor non-prod shared AI services in nonprod-shared if applicable.
15. Add prod Front Door inside prod.
16. Add AKS, networking, identities, monitoring, database, storage, and secrets resources per environment as applicable.
17. Add kGateway bootstrap.
18. Add ArgoCD bootstrap.
19. Create or refactor Helm chart.
20. Create GitOps desired-state layout.
21. Create CI/CD workflow files.
22. Validate Terraform formatting.
23. Validate Terraform for each env.
24. Plan Terraform for each env.
25. Validate Helm lint.
26. Validate Helm template for dev/staging/prod.
27. Validate GitOps YAML.
28. Validate CI/CD workflow syntax.
29. Build Docker images where applicable.
30. Confirm no references to deleted paths remain.
31. Clean up old infra/deployment files classified as safe DELETE.
32. Leave REVIEW items documented.
33. Update all context docs and final reports.

Validation commands:

* terraform fmt -recursive
* terraform validate for each env
* terraform plan for global-shared
* terraform plan for nonprod-shared
* terraform plan for dev
* terraform plan for staging
* terraform plan for prod
* helm lint
* helm template with dev values
* helm template with staging values
* helm template with prod values
* validate GitOps YAML
* validate GitHub Actions workflow syntax
* build Docker images where applicable
* run repository-wide reference scans for removed paths

Final deliverables:

1. FINAL_REFACTOR_SUMMARY.md

Include:

* What changed.
* New structure.
* Infrastructure changes.
* Helm changes.
* GitOps changes.
* CI/CD changes.
* Deleted files/folders.
* Remaining manual review items.
* Validation status.
* Known risks.
* Next steps.

2. FINAL_FOLDER_STRUCTURE.md

Include:

* Final tree structure.
* Purpose of each folder.
* Which folders are future standalone repos.
* Which files are required.
* Which old structures were removed.

3. FINAL_CLEANUP_REPORT.md

Include:

* Removed redundant files.
* Kept required files.
* Merged files.
* Files needing manual review.
* Validation evidence.
* Remaining cleanup TODOs.

4. FINAL_OPERATIONS_GUIDE.md

Include:

* How to run locally.
* How to build images.
* How to deploy infra.
* Terraform apply order.
* How GitOps deployment works.
* How to promote dev to staging to prod.
* How to roll back.
* How to handle secrets.
* How to troubleshoot common issues.

Important:
Do not leave old and new deployment systems active at the same time unless documented.
Do not leave stale duplicate folders.
Do not leave unused root-level files.
Do not leave old workflows that can accidentally deploy.
Do not leave old Terraform roots that can accidentally manage the same infrastructure.
Do not delete Terraform state files, secrets, certificates, kubeconfigs, or unknown production files.
The final result must be clean, minimal, validated, and ready for future repo splitting.
