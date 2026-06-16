# AKS Raw YAML

Raw Kubernetes manifests live in `repos/spendpilot-infra/k8s/`.

Apply order:

1. `namespace.yaml`
2. `serviceaccount.yaml`
3. `configmap.yaml`
4. `secret.example.yaml` after replacing placeholders
5. `migration-job.yaml`
6. deployments and services
7. `gateway.yaml`
8. `httproute-*.yaml`
9. optional `hpa-*.yaml`

The Helm chart is the preferred production path. Raw YAML exists for review, debugging, and low-level troubleshooting.
