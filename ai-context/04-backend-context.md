# Backend context

The backend is a shared FastAPI codebase with multiple entrypoints:

- `app.main:app`
- `app.service_apps.identity:app`
- `app.service_apps.finance:app`
- `app.service_apps.documents:app`

Important modules:

- `app/core/config.py`
- `app/core/security.py`
- `app/core/rbac.py`
- `app/models/__init__.py`
- `app/services/user_service.py`
- `app/services/finance_service.py`
- `app/services/document_service.py`
- `app/services/ai_foundry_service.py`
- `app/services/ai_chat_service.py`

Key responsibilities:

- validate Entra tokens
- bootstrap organizations from Entra tenant claims or personal-account identities
- persist user sessions
- enforce RBAC
- manage budgets, expenses, approvals, and audit data
- manage recurring expenses, recurring requests, spend limits, and payment priorities
- extract OCR text and invoice fields
- analyze documents with Azure AI Foundry
- persist tenant-scoped AI chat sessions and grounded assistant messages

2026-06-15 Phase 1 note:

- membership roles are being canonicalized toward `org_owner`, `dept_head`, and `employee`
- organization-scoped departments and onboarding state now live on `organization_memberships`
- auth and admin routes now expose department listing, onboarding completion, and org-owner member-management updates

2026-06-15 continued refactor note:

- the backend now exposes role-aware payment-operations APIs for variable expenses, recurring expenses, recurring requests, spend limits, payment priorities, and AI chat
