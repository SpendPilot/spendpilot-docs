# Refactor log

Latest major refactor:

1. validated the repo and docs against the current runtime
2. expanded the backend from a small document scanner into a tenant-aware finance platform
3. introduced organizations, memberships, sessions, budgets, expenses, approvals, and audit events
4. split the AKS backend runtime into identity, finance, and documents services
5. upgraded document processing to include invoice extraction
6. rewired Helm and raw manifests to use Gateway API HTTPRoutes
7. replaced the AKS Terraform path with Front Door + WAF + kGateway + Workload Identity bootstrap
8. updated README, deployment docs, and AI context files
9. corrected Entra API scope wiring to use the backend Application ID URI
10. refactored auth bootstrap so personal Microsoft accounts get isolated workspaces instead of being rejected or merged together
11. hardened frontend API URL building so `/api` base paths cannot turn into `/api/api/...` during login callbacks
12. bundled Entra consent between the frontend SPA and backend API with known clients and pre-authorization, and added an admin-consent path for external customer tenants
13. forced explicit account selection on Microsoft login so cached browser sessions do not silently reuse the wrong tenant user
14. fixed document extraction persistence so Decimal invoice totals from Document Intelligence or Foundry do not trigger PostgreSQL JSON serialization errors
15. corrected auth bootstrap so guest Microsoft accounts inside a real Entra tenant join that tenant workspace instead of being split into separate personal workspaces
16. tested a Front Door HTTPS-origin path with a Terraform-managed gateway TLS secret, then finalized the validated default on HTTP origin plus an Azure cloudapp FQDN override because Front Door would not reliably accept the self-signed origin certificate
17. added a Front Door WAF auth rate-limit rule on `/api/auth`
18. uplifted PostgreSQL from Burstable to `GP_Standard_D2s_v3` and enabled `ZoneRedundant` HA plus geo-redundant backup
19. kept Blob Storage on OAuth-first auth with local users disabled, while re-enabling shared-key auth only to preserve Terraform/AzureRM storage-account management compatibility
20. modeled the PostgreSQL subnet storage service endpoint in Terraform so the live deployment returns a clean plan after the HA hardening pass
21. switched the stable Front Door origin target to the gateway public IP's Azure cloudapp FQDN and modeled the validated apex-plus-dedicated-www custom-domain routing shape back into Terraform
22. added Phase 1 product-refactor identity changes: canonical `org_owner` / `dept_head` / `employee` membership roles, default departments, employee onboarding state, and org-owner membership management guardrails
23. extended the finance data model and APIs for recurring expenses, recurring requests, spend limits, payment priorities, AI chat sessions, and richer budget / bill linkage
24. converted the existing frontend into a role-aware payment-operations baseline with org-owner, dept-head, and employee workspaces
25. converted the backend deployment split from one shared image with runtime `APP_MODULE` overrides into three service folders with dedicated Dockerfiles and image repositories

Intentional simplification kept:

- local docker compose still uses the combined backend entrypoint for easier development
