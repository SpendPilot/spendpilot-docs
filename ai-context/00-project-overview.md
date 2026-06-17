# Project overview

This repository is the deployable monorepo for the Spend Control Platform.

Current deployable units:

- `repos/spendpilot-frontend/`: Next.js finance workspace
- `repos/spendpilot-services/`: shared FastAPI package with one combined local app plus dedicated `identity`, `finance`, and `documents` service folders
- `repos/spendpilot-helm/charts/spendpilot/`: canonical Kubernetes application chart
- `repos/spendpilot-infra/k8s/`: raw manifest mirror
- `repos/spendpilot-infra/envs/dev/`: current active Azure, Entra, AKS, Front Door, and Helm bootstrap root

Business scope:

- multi-tenant SaaS for both Entra organizations and personal Microsoft account users
- expenses, approvals, budgets, and finance documents
- OCR and invoice extraction
- AI-assisted finance review
