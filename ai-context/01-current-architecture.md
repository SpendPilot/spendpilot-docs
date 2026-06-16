# Current architecture before refactor

Repository scan findings from the original project state:

- Frontend lived in `../spend-control-frontend`
- Public API and orchestration lived in `../spend-control-control-service`
- Expense domain logic lived in `../spend-control-expense-service`
- AI workflows and Ollama integration lived in `../spend-control-ai-service`
- Platform deployment files lived here in `k8s/`, `docker-compose*.yml`, and `terraform/`

Notable patterns found:

- The frontend called `control-service` directly through `NEXT_PUBLIC_API_BASE_URL`
- `control-service` delegated to `expense-service` and `ai-service`
- Authentication was demo-oriented username/password plus opaque tokens stored in localStorage
- AI features used an internal Ollama service and prompt files instead of Azure AI services
- Kubernetes files used an Ingress resource at `k8s/ingress.yaml`

Business features found in the old app:

- Expense submission
- Manager approvals
- Policy rule management
- Budget overview
- Anomaly inbox
- AI explanations and chat

Why this mattered:

- The codebase was more complex than the target product needs
- Authentication was not ready for business tenant sign-in
- Deployment assets were aligned to the old three-service topology
