# AKS Helm Deployment

The application Helm chart deploys the full AKS runtime shape:

- frontend
- identity-service
- finance-service
- documents-service
- migration job
- Gateway and HTTPRoutes

Render:

```bash
helm template spend-control repos/spendpilot-helm/charts/spendpilot
```

Install:

```bash
helm upgrade --install spend-control repos/spendpilot-helm/charts/spendpilot \
  --namespace spend-control \
  --create-namespace \
  -f repos/spendpilot-helm/charts/spendpilot/values-prod.yaml
```

Terraform already drives this release in the AKS stack, so manual Helm is mainly for local validation and emergency operations.
