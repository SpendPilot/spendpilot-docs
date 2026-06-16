# AKS, Front Door, WAF, and kGateway

This repository now targets:

```txt
User
  -> Azure Front Door Premium
  -> WAF policy
  -> kGateway service on AKS
  -> Gateway API HTTPRoutes
  -> frontend / identity / finance / documents services
```

Routing rules:

- `/` -> `spend-control-frontend`
- `/api/auth`, `/api/admin`, `/health`, `/ready` -> `spend-control-identity`
- `/api/finance` -> `spend-control-finance`
- `/api/documents`, `/api/ai` -> `spend-control-documents`

Operational notes:

- Front Door probes `http://<origin>/health`
- kGateway is the only public AKS entrypoint
- Backend services stay `ClusterIP`
- Browser traffic should keep `NEXT_PUBLIC_API_BASE_URL=/api`
- Front Door default domain is used by default; custom domains can be added later
- The kGateway chart is vendored in `repos/spendpilot-infra/vendor/kgateway/` to avoid OCI pull issues in restricted networks
- The app namespace is created by Terraform before the Helm release; the chart itself should not also be relied on for namespace bootstrap
- The migration job is a post-install and post-upgrade hook because it depends on the chart-created ServiceAccount and Secret
- The AKS gateway service is a `LoadBalancer` service named `spend-control-gateway`
- The AKS gateway service exposes `80` in the validated default path
- Terraform can still create a gateway TLS secret, but the validated default keeps Front Door on `HTTP` to avoid self-signed origin certificate failures
- Front Door uses the gateway public IP's Azure cloudapp FQDN as the origin target when available
- Front Door may take additional time to propagate even after `provisioningState` is `Succeeded`
- The validated route keeps browser HTTPS at the Front Door edge while forwarding `HTTP` to the AKS gateway origin
