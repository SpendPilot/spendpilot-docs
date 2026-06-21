# SpendPilot AI Agent Knowledge Base

This curated knowledge base is intentionally small and maintainable.

## What SpendPilot Is

SpendPilot is a finance operations platform for budgets, approvals, variable expenses, recurring expenses, payment prioritization, and document intake.

## Roles

- `org_owner`: organization-wide visibility and final approval authority
- `dept_head`: department-scoped visibility and approval responsibilities
- `employee`: self-scoped visibility with normal submission workflows

## Core Finance Concepts

- Variable expenses are submitted operational spends.
- Recurring expenses represent repeating vendor obligations.
- Company budgets define the monthly organization envelope.
- Department budgets must fit within the company budget.

## Approval Flow

- Employee variable expenses begin at department-head review.
- Department heads can approve, reject, or forward items.
- org_owner performs final approval when policy thresholds or escalation require it.

## Documents

- Uploaded documents can be scanned for OCR, invoice extraction, and risk analysis.
- The chat assistant should prefer document metadata and stored scan summaries.
- Raw document excerpts must stay short and permission-safe.

## Assistant Boundaries

- No vector database is used.
- The assistant must stay grounded in curated knowledge plus bounded internal tools.
- It must not invent finance numbers or access cross-tenant data.
- It must not generate SQL or expose secrets.
