PROJECT_NAME: SpendPilot

You are a senior software architect, refactoring engineer, Azure DevOps engineer, and platform engineer.

Before making changes, read these context files:

* AI_CONTEXT.md
* CURRENT_STATE.md
* TARGET_ARCHITECTURE.md
* DECISIONS.md
* REFACTOR_PLAN.md
* RISK_REGISTER.md
* MIGRATION_CHECKLIST.md
* CLEANUP_STRATEGY.md

Goal:
Refactor the application and repository structure so the project is cleanly organized, microservice-ready, and ready for CI/CD and infrastructure separation.

Important:

* Preserve current business behavior.
* Do not change working logic.
* Do not rewrite features unnecessarily.
* Keep backend services grouped for now.
* Make each service folder ready to become an independent repository later.
* Clean up redundant files only after validation.
* The final structure should contain only required files.
* Do not touch Terraform state or cloud resources in this phase.

Use this target repo strategy:

repos/
<PROJECT_NAME>-frontend/
<PROJECT_NAME>-services/
<PROJECT_NAME>-helm/
<PROJECT_NAME>-infra/
<PROJECT_NAME>-gitops/
<PROJECT_NAME>-docs/

Do not require remote Git repositories to exist yet.
Prepare the folders so each one can become its own repo later.

Application target:

1. Frontend

Move frontend code into:
repos/<PROJECT_NAME>-frontend/

Include only required frontend files:

* source code
* package files
* lock files
* public/static assets
* frontend config files
* frontend tests
* frontend Dockerfile if needed
* frontend-specific README
* frontend local environment template files
* frontend CI workflow only if temporarily colocated

Do not include:

* backend files
* Terraform files
* Helm charts
* old deployment manifests
* duplicate README files
* generated build output
* stale configs
* unrelated docs

2. Backend/services

Move backend code into:
repos/<PROJECT_NAME>-services/

Target structure:

repos/<PROJECT_NAME>-services/
services/ <service-1>/ <service-2>/ <service-3>/ <service-4>/ <service-5>/ <service-6>/ <service-7>/
libs/
common/
config/
auth/
observability/

The service count must be based on the real project.
Prefer 5-7 service boundaries only if natural.
Do not force artificial services.
If the current code is tightly coupled, create fewer clean service domains first and document future splits.

Each service should have:

* README.md
* clear entrypoint
* Dockerfile if it is deployed independently
* tests folder if applicable
* config example
* dependency file if needed
* clear ownership of its domain logic
* no circular dependency on another service folder

Shared libraries may contain only technical utilities:

* logging
* config loading
* common errors
* auth middleware helpers
* common DTOs/types only when necessary
* observability helpers

Do not move business logic into shared libraries.

3. Docs

Move documentation into:
repos/<PROJECT_NAME>-docs/

Keep only useful docs:

* architecture docs
* runbooks
* setup docs
* deployment docs
* troubleshooting docs
* diagrams
* current project-specific notes

Stale, duplicated, generated, or outdated docs must be classified in CLEANUP_PLAN.md before removal.

4. Helm, infra, and GitOps preparation

Do not fully implement infrastructure in this phase.
Only prepare or move obvious folders when safe:

* Helm charts to repos/<PROJECT_NAME>-helm/
* Terraform/infrastructure to repos/<PROJECT_NAME>-infra/
* ArgoCD/GitOps manifests to repos/<PROJECT_NAME>-gitops/

Infrastructure details will be finalized in the next phase.

Service-boundary process:

1. Inspect current backend structure.
2. Identify actual business domains.
3. Propose service boundaries.
4. Create SERVICE_BOUNDARIES.md.
5. Create SERVICE_SPLIT_READINESS.md.
6. Create REPO_SPLIT_PLAN.md.
7. Move code incrementally.
8. Fix imports.
9. Fix package paths.
10. Fix local dev commands.
11. Fix Dockerfiles.
12. Fix tests.
13. Validate.
14. Classify old locations.
15. Only then clean up safe old locations.

Create/update these files:

1. SERVICE_BOUNDARIES.md

Include:

* Proposed services.
* What each service owns.
* What code moved into each service.
* What remains shared.
* API boundaries.
* Database/data ownership if discoverable.
* Current coupling between services.
* Future split strategy.
* Suggested future repo name per service.

2. SERVICE_SPLIT_READINESS.md

For each service include:

* Current readiness to become a separate repo.
* Blockers.
* Shared dependencies.
* Database dependency.
* API dependency.
* Required future work.
* Suggested independent repo name.
* Suggested future CI workflow.

3. LOCAL_DEV.md

Include:

* How to run frontend.
* How to run backend/services.
* How to run all services together.
* How to run with Docker Compose if applicable.
* Required environment variables.
* Required local dependencies.
* Database setup.
* Storage/blob setup if applicable.
* AI service setup if applicable.
* Common troubleshooting.

4. REPO_SPLIT_PLAN.md

Include:

* Current monorepo layout.
* Target repo layout.
* What moves where.
* What stays temporarily.
* What must be deleted after validation.
* Suggested future Git remote names.
* Future split order.
* Which folders are already repo-ready.
* Which folders still depend on monorepo assumptions.

5. CLEANUP_PLAN.md

Build a cleanup table with columns:

* Path
* Classification: KEEP / MOVE / MERGE / DELETE / REVIEW
* Reason
* Replacement path
* References found
* Safe to remove now: yes/no
* Validation required before removal

Cleanup rules:

1. Before deleting a file or folder, search for references to it.
2. Use repository-wide searches for imports, paths, Docker references, package references, Terraform references, Helm references, Kubernetes references, CI/CD references, docs references, and script references.
3. Do not delete files that are still referenced.
4. Do not delete secrets, certificates, tfstate files, tfvars with real secrets, kubeconfigs, production configs, or unknown credential-like files.
5. Flag sensitive or unknown files in CLEANUP_PLAN.md instead of deleting them.
6. Remove duplicate files only after confirming the replacement exists.
7. Remove old folders only after the new folder builds/tests successfully.
8. Remove obsolete generated files, cache folders, old build outputs, dead examples, duplicate manifests, stale Dockerfiles, old Helm values, old workflow files, and unused scripts only after validation.
9. Keep .gitignore updated so generated files do not return.
10. The final structure must not contain both old and new copies of the same active code.
11. The final structure must contain only required files.
12. If unsure whether a file is needed, classify it as REVIEW instead of deleting it.

Suggested cleanup candidates:

* old app folders after migration
* duplicate Dockerfiles
* duplicate docker-compose files
* old Kubernetes manifests replaced by Helm/GitOps
* old Helm values no longer used
* old Terraform workspace files
* old environment folders no longer used
* old CI/CD workflows replaced by new workflows
* generated build output
* caches
* unused scripts
* stale docs
* empty folders
* duplicate README files
* obsolete local config files

After refactor, create:

6. CLEANUP_REPORT.md

Include:

* Files/folders kept.
* Files/folders moved.
* Files/folders merged.
* Files/folders deleted.
* Files/folders requiring manual review.
* Validation performed.
* Known remaining cleanup TODOs.
* Any duplicate structures intentionally kept and why.

Validation requirements:

Run or document the exact commands for:

* frontend install
* frontend build
* frontend tests
* backend install
* backend tests
* service-level tests
* service-level Docker builds
* lint/typecheck if available
* local run command
* Docker Compose command if available
* repository-wide reference scan for deleted paths

Update:

* AI_CONTEXT.md
* DECISIONS.md
* MIGRATION_CHECKLIST.md
* RISK_REGISTER.md
* CLEANUP_STRATEGY.md

Important:
Make changes incrementally.
After each major move, verify references.
Do not leave duplicate active folders.
Do not leave stale root-level files unless intentionally required.
Do not leave old app folders active after the new structure is validated.
When unsure whether a file is needed, classify it as REVIEW instead of deleting it.
At the end, provide a clear summary of what changed, what was validated, what was deleted, and what still needs manual review.
