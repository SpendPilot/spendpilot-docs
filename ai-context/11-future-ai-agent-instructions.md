# Future AI agent instructions

When continuing work in this repo:

- treat this repository as the only supported runtime source
- preserve the Front Door -> kGateway -> HTTPRoute topology
- preserve the small 3-service backend split unless there is a strong operational reason to change it
- preserve the dedicated service folders and service-specific container images under `repos/spendpilot-services/services/`
- do not reintroduce Kubernetes Ingress as the default path
- keep Workload Identity and `DefaultAzureCredential` as the preferred Azure auth model
- update `repos/spendpilot-docs/ai-context/` whenever the runtime shape, auth model, or deployment path changes
- preserve the vendored kGateway charts unless OCI pulls become reliable again in the target environment
- keep the app namespace bootstrap in Terraform, not inside the app Helm release
- keep the migration job as a post-install and post-upgrade hook
- prefer GitHub-backed ACR Tasks when the operator workstation is constrained by SSL interception or Docker restrictions
- treat prod AKS as a private cluster reached operationally through the Australia ops jumpbox and Azure control-plane workflows
- do not reintroduce a direct Kubernetes-provider dependency in `repos/spendpilot-infra/envs/prod`
- preserve the CLI-backed in-place AKS orchestration in prod unless the native Terraform provider can prove no-replacement behavior for the same operations
- preserve Azure managed Prometheus and Azure Managed Grafana as the default prod observability path

Likely next extensions:

- background job processing for long-running scans
- enterprise onboarding automation for customer tenant consent
- richer approval policy workflows
- custom domains and private-link style edge hardening
