# Target architecture

```txt
User
  -> Azure Front Door Premium + WAF
  -> HTTPS to the kGateway service on AKS
  -> HTTPRoutes
      -> frontend
      -> identity-service
      -> finance-service
      -> documents-service
  -> PostgreSQL Flexible Server (General Purpose, zone-redundant HA, geo-backup)
  -> Azure Blob Storage
  -> Azure AI Foundry
  -> Azure AI Document Intelligence
```

Target deployment characteristics:

- multi-tenant Entra SaaS
- one system node pool and one user node pool
- `Standard_D2s_v3` nodes in the validated subscription
- `GP_Standard_D2s_v3` PostgreSQL in the validated subscription
- Workload Identity with one user-assigned managed identity
- Helm-managed application deployments
