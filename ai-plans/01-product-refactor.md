PROJECT_NAME: <PROJECT_NAME>

Assume the following standard platform architecture unless the repository clearly proves otherwise.

Cloud/platform:

* Azure

Infrastructure:

* Terraform
* No Terraform workspaces for environment separation
* Use separate env folders and separate backend state files
* Use Azure Blob Storage remote backend for Terraform state

Kubernetes/platform:

* AKS
* ACR
* kGateway for in-cluster Gateway API routing
* ArgoCD for GitOps-based CD

CI/CD:

* GitHub Actions
* Azure authentication through OIDC where possible
* CI builds, tests, scans, builds Docker images, and pushes to ACR
* CD happens through GitOps/ArgoCD, not direct production Helm upgrades

Environments:

* dev
* staging
* prod

Target repository structure:

* <PROJECT_NAME>-frontend
* <PROJECT_NAME>-services
* <PROJECT_NAME>-helm
* <PROJECT_NAME>-infra
* <PROJECT_NAME>-gitops
* <PROJECT_NAME>-docs

Application strategy:

* Keep backend services grouped for now.
* Structure services so each microservice can later become its own independent repository.
* Target 5-7 microservices only if the current codebase naturally supports it.
* Do not force unnecessary service splits.
* Preserve existing business logic and working behavior.

Shared-resource strategy:

* global-shared: resources shared by all environments, especially ACR and truly global/common resources.
* nonprod-shared: resources shared by dev and staging, such as non-prod Front Door, non-prod WAF, shared non-prod AI services, and other cost-saving shared non-prod resources.
* dev: dev runtime resources.
* staging: staging runtime resources.
* prod: prod runtime resources and prod Front Door.

Production strategy:

* Prod Front Door must live inside envs/prod.
* Prod AKS should be public for now.
* Secure public prod AKS with reasonable controls such as authorized IP ranges where possible, Azure RBAC, managed identity, workload identity, no long-lived secrets, and clear future TODOs for private AKS.
* Use private endpoints/private DNS where useful, but do not overengineer.

Terraform deployment strategy:

* Terraform provisions Azure infrastructure and platform bootstrap components.
* Terraform may bootstrap AKS, kGateway, and ArgoCD.
* Terraform must not deploy application microservices directly.
* Application workloads must be deployed through ArgoCD/GitOps.

Cleanup strategy:

* After refactoring and validation, remove old redundant folders/files.
* Do not leave duplicate active structures.
* Do not leave old workflows that can accidentally deploy.
* Do not leave old Terraform roots that can accidentally manage the same infrastructure.
* Classify leftover files as KEEP / MOVE / MERGE / DELETE / REVIEW before removal.
* Search references before deleting anything.
* Never delete state files, secrets, credentials, certificates, kubeconfigs, tfvars containing real values, or unknown production files automatically.

You are a senior software architect, Azure DevOps engineer, cloud infrastructure engineer, Terraform engineer, Kubernetes engineer, GitOps engineer, CI/CD engineer, and professional refactoring engineer.

I am doing a major refactor of this project.

Current state:

* The current project may be a monorepo.
* It may contain frontend code, backend code, microservices, docs, Terraform/infrastructure code, Helm charts, Kubernetes manifests, GitHub Actions workflows, scripts, and old unused files.
* The project may already be working, so the existing business behavior must be preserved.

Your first task is discovery and context engineering only.

Do not immediately rewrite the project.
Do not delete files.
Do not move large folders.
Do not change business logic.
Do not break the currently working application.
Do not perform destructive operations.

Your goal is to inspect the repository deeply and create persistent context files so future AI/Codex sessions can continue the refactor safely.

When future phases are executed, they must read these context files first before making changes.

If a detail is missing, inspect the repository and make the safest assumption. Only ask questions if a missing value truly blocks progress.

Critical rules:

1. Preserve existing business and functional behavior.
2. Do not change business logic unless absolutely necessary.
3. Refactor structure, ownership, deployment, and CI/CD safely.
4. Prefer incremental migration over a dangerous big-bang rewrite.
5. Keep backend services grouped for now.
6. Design each service so it can later become an independent repository.
7. Remove redundant files only in a later cleanup phase after validation.
8. Never delete secrets, Terraform state files, certificates, kubeconfigs, environment-specific files, or unknown production files without clearly flagging them first.
9. Do not commit or expose secrets.
10. Keep naming simple, readable, and based on PROJECT_NAME.

Create or update these context files:

1. AI_CONTEXT.md

Include:

* What the project appears to be.
* Current architecture.
* Target architecture.
* Current technologies found.
* Important constraints.
* What future AI sessions must preserve.
* What must not be changed.
* The standard Azure/Terraform/AKS/ACR/kGateway/ArgoCD/GitHub Actions direction.
* A reminder that future sessions must read this file before changing code.

2. CURRENT_STATE.md

Include:

* Current folder structure.
* Purpose of each major folder.
* Frontend location.
* Backend/service location.
* Infrastructure location.
* Helm/Kubernetes manifest location.
* CI/CD location.
* Docs location.
* Scripts location.
* Current build/test/run/deploy flow if discoverable.
* Hardcoded paths.
* Cross-folder dependencies.
* Environment variables found.
* Files that look duplicated, stale, generated, or unused.
* Files that look sensitive and must not be deleted automatically.

3. TARGET_ARCHITECTURE.md

Include:

* Proposed target repo/folder structure.
* Proposed service boundaries.
* Proposed infrastructure layout.
* Proposed Helm/GitOps layout.
* Proposed CI/CD layout.
* Proposed environment model.
* Proposed shared-resource model.
* How the project can later split into multiple repos safely.
* How the backend services can later become independent repos.
* How Terraform, Helm, GitOps, and application code should be separated.

4. DECISIONS.md

Record decisions using this style:

* Decision:
* Reason:
* Impact:
* Future change option:

Include decisions for:

* Project naming.
* Service grouping.
* Future microservice split readiness.
* Terraform/environment layout.
* Removing Terraform workspaces.
* Shared resources.
* Non-prod shared resources.
* Prod Front Door inside envs/prod.
* Public prod AKS for now.
* GitOps/CD approach.
* Cleanup rules.
* Naming scheme.

5. REFACTOR_PLAN.md

Create a phased plan:

* Phase 1: discovery and context files.
* Phase 2: application/service boundary refactor.
* Phase 3: repo/folder split preparation.
* Phase 4: infrastructure refactor.
* Phase 5: Helm/GitOps refactor.
* Phase 6: CI/CD setup.
* Phase 7: cleanup and removal of redundant files.
* Phase 8: validation and migration checklist.

For each phase include:

* Goal.
* Files likely affected.
* Risks.
* Validation commands.
* Rollback notes.

6. RISK_REGISTER.md

Include risks and mitigations for:

* Broken imports.
* Broken package paths.
* Broken Dockerfiles.
* Broken Docker Compose.
* Broken Helm values.
* Broken Kubernetes manifests.
* Broken Terraform file paths.
* Terraform state mistakes.
* Shared resource dependency issues.
* CI/CD workflow breakage.
* Secret leakage.
* Duplicate environment configs.
* Dead files left after refactor.
* Removing files that are still referenced.
* Old workflows accidentally deploying.
* Multiple Terraform states managing the same resource.

7. MIGRATION_CHECKLIST.md

Include:

* Pre-refactor backup steps.
* Branching strategy.
* Commit strategy.
* Validation commands.
* Step-by-step migration checklist.
* Rollback notes.
* Final cleanup checklist.
* Manual review checklist.

8. CLEANUP_STRATEGY.md

Include:

* How to identify files to keep.
* How to identify files to move.
* How to identify files to merge.
* How to identify files to delete.
* How to verify that deleted files are not referenced.
* Rules for generated files, old configs, old workflows, old Helm values, old Terraform files, docs, scripts, caches, and local-only files.
* Rules for classifying files as KEEP / MOVE / MERGE / DELETE / REVIEW.
* Rules for secrets, state files, certificates, kubeconfigs, production configs, and unknown sensitive files.
* Reference scanning strategy before removal.

After creating these files, provide:

1. Summary of current architecture.
2. Proposed target architecture.
3. Main risky areas.
4. Recommended next step.
5. List of missing information only if truly blocking.

Do not delete files in this phase.
Do not move large folders in this phase.
Do not perform destructive operations in this phase.
