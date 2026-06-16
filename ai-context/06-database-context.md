# Database context

Primary schema now includes:

- `organizations`
- `users`
- `organization_memberships`
- `departments`
- `user_sessions`
- `expense_categories`
- `budgets`
- `expenses`
- `expense_approvals`
- `vendors`
- `recurring_expenses`
- `recurring_expense_requests`
- `spend_limits`
- `payment_priorities`
- `documents`
- `document_scans`
- `ai_chat_sessions`
- `ai_chat_messages`
- `audit_events`

Database expectations:

- SQLite stays acceptable for tests
- PostgreSQL Flexible Server is the intended Azure runtime
- one logical database serves all three backend services
- `organizations.tenant_id` stores either an Entra tenant ID or a synthetic `msa:<hash>` key for personal Microsoft accounts

2026-06-15 Phase 1 note:

- `organization_memberships` now carries department assignment and onboarding completion state
- `departments` is organization-scoped and currently seeded with `IT`, `Marketing`, and `HR`

2026-06-15 continued refactor note:

- the existing `expenses` table is now the variable-expense baseline
- the existing `documents` table is now the bills-library baseline with added linked-expense metadata
