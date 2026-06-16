# API map

Shared health:

- `GET /health`
- `GET /ready`

Identity service:

- `GET /api/auth/me`
- `GET /api/auth/departments`
- `POST /api/auth/onboarding/department`
- `POST /api/auth/dev-login`
- `GET /api/admin/organization`
- `GET /api/admin/departments`
- `GET /api/admin/members`
- `PATCH /api/admin/members/{membership_id}`
- `GET /api/admin/sessions`
- `POST /api/admin/sessions/{session_id}/revoke`

Finance service:

- `GET /api/finance/dashboard`
- `GET /api/finance/categories`
- `GET /api/finance/budgets`
- `POST /api/finance/budgets`
- `GET /api/finance/expenses`
- `GET /api/finance/expenses/variable`
- `POST /api/finance/expenses/variable`
- `POST /api/finance/expenses/{expense_id}/forward`
- `POST /api/finance/expenses/{expense_id}/approve`
- `POST /api/finance/expenses/{expense_id}/reject`
- `POST /api/finance/expenses/{expense_id}/paid`
- `GET /api/finance/recurring-expenses`
- `POST /api/finance/recurring-expenses`
- `PATCH /api/finance/recurring-expenses/{recurring_id}`
- `GET /api/finance/recurring-expense-requests`
- `POST /api/finance/recurring-expense-requests`
- `POST /api/finance/recurring-expense-requests/{request_id}/decision`
- `GET /api/finance/spend-limits`
- `POST /api/finance/spend-limits`
- `PATCH /api/finance/spend-limits/{spend_limit_id}`
- `GET /api/finance/payment-priorities`
- `GET /api/finance/audit-events`

Documents service:

- `GET /api/documents`
- `POST /api/documents/upload`
- `GET /api/documents/{id}`
- `POST /api/documents/{id}/scan`
- `GET /api/documents/{id}/scan-result`
- `POST /api/documents/{id}/extract-expense`

AI service:

- `POST /api/ai/analyze`
- `GET /api/ai/sessions`
- `POST /api/ai/chat`
