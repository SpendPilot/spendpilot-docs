# Current AI Service Handoff

This document describes exactly how the current SpendPilot AI service works today, so another agent can safely refactor it without guessing.

## Update Note

As of the latest AI assistant refactor:

- `/api/ai/chat` now prefers an Azure AI Foundry-backed, tool-grounded assistant path.
- The legacy keyword-based finance reply builder still exists as the fallback path if the model or tool flow fails.
- No vector database was introduced.
- Tenant-scoped finance/document tools and a small curated knowledge base now ground the assistant response.

## Scope

There are two separate AI-related paths in the current system:

1. Chat-style AI insights for finance users via `/api/ai/chat`
2. Document OCR, invoice extraction, and document risk analysis via the documents service

These two paths are very different.

The most important facts are now:

- The chat assistant prefers Azure OpenAI / Foundry when configured.
- The chat assistant still preserves the old deterministic grounded fallback path.
- The document analysis flow continues to use Azure AI Foundry and Azure Document Intelligence.

## Repositories And Key Files

### Backend

- `spendpilot-services/app/api/routes/ai.py`
- `spendpilot-services/app/services/ai_chat_service.py`
- `spendpilot-services/app/services/ai_foundry_service.py`
- `spendpilot-services/app/services/document_service.py`
- `spendpilot-services/app/services/policy_service.py`
- `spendpilot-services/app/core/config.py`
- `spendpilot-services/app/schemas/ai.py`
- `spendpilot-services/app/schemas/documents.py`
- `spendpilot-services/app/models/__init__.py`

### Frontend

- `spendpilot-frontend/app/ai-insights/page.tsx`

### Deployment/config wiring

- `spendpilot-helm/charts/spendpilot/templates/configmap.yaml`
- `spendpilot-gitops/environments/dev/values/spendpilot-values.yaml`
- `spendpilot-gitops/environments/prod/values/spendpilot-values.yaml`

## High-Level Architecture

### Path 1: AI chat/insights

Request flow:

1. Frontend page `ai-insights` loads existing sessions from `/api/ai/sessions`
2. Frontend posts user text to `/api/ai/chat`
3. Backend stores the user message
4. Backend generates a reply using `AIChatService._build_grounded_reply(...)`
5. Backend stores the assistant reply
6. Frontend reloads the session and renders the message history

Important:

- This path is now tool-grounded when Foundry is configured.
- It uses a curated knowledge base plus bounded internal tools for finance and document lookups.
- It preserves the deterministic reply builder as fallback for resilience.

### Path 2: Document AI

Request flow:

1. User uploads a document through the documents flow
2. `DocumentService` reads the file bytes from storage
3. `AIFoundryService` extracts text
4. `AIFoundryService` tries to extract structured expense fields
5. `AIFoundryService` analyzes the document for risk/policy findings
6. Results are stored as `DocumentScan`
7. Optional extracted expense fields are copied onto an associated expense record

Important:

- This path uses Azure Document Intelligence for OCR and invoice extraction when available.
- This path uses Azure AI Foundry/OpenAI for JSON-formatted extraction and analysis when enabled.
- This path has a fallback analyzer when AI services are unavailable.

## Path 1 In Detail: Current Chat Assistant

### API routes

Defined in `spendpilot-services/app/api/routes/ai.py`.

Current endpoints:

- `POST /api/ai/analyze`
- `GET /api/ai/sessions`
- `POST /api/ai/chat`

### Frontend behavior

Defined in `spendpilot-frontend/app/ai-insights/page.tsx`.

Current behavior:

- On load, the page calls `/api/ai/sessions`
- The default input text is `"How is my cash flow this month?"`
- On submit, the page posts:
  - `message`
  - `session_id: sessions[0]?.id || null`
- After sending, it reloads the sessions and displays the first session's messages

Current limitations in the frontend:

- No streaming
- No typing indicator beyond a basic button state
- No session selector UI
- No prompt presets UI
- No source citations
- No structured answer cards
- No explicit reasoning mode or document-aware mode

### Data model

Defined in `spendpilot-services/app/models/__init__.py`.

Relevant tables/models:

- `AIChatSession`
- `AIChatMessage`

Stored data:

- session title
- user/organization ownership
- message role
- message text
- optional `grounded_context_json`
- timestamps

This means the chat history is persisted in the app database, but the current assistant is still deterministic and rule-based.

### Current backend logic

Defined in `spendpilot-services/app/services/ai_chat_service.py`.

The core method is:

- `AIChatService._build_grounded_reply(db, principal, message)`

This function:

1. Loads tenant-scoped finance dashboard data
2. Recomputes payment priorities
3. Loads top recurring payments
4. Counts approved but unpaid expenses
5. Runs keyword matching against the user message
6. Assembles a canned text response from the data

### Exact data sources used by the chat assistant

The reply is grounded in:

- `FinanceService.build_dashboard(db, principal)`
- `FinanceService.recalculate_payment_priorities(db, principal)`
- `RecurringExpense` query for active items ordered by next due date
- `Expense` query for approved but unpaid variable expenses

### Exact response behavior

The current assistant checks the lowercased message for keywords and builds response text from those branches.

Current keyword branches:

- `"cash flow"` or `"outflow"`
  - returns weekly and monthly cash outflow

- `"overspend"` or `"budget"`
  - returns company budget used and remaining

- `"department"`
  - returns the highest-spend department

- `"urgent"` or `"pay"`
  - returns the most urgent payment priority

- `"pending"` or `"approve"`
  - returns pending approvals and approved-unpaid expense counts

- `"recurring"`
  - returns a short list of upcoming recurring payments

If no keywords match:

- it returns a generic monthly spend summary
- mentions pending approvals
- optionally mentions highest-spend department
- optionally mentions top priority payment
- adds a budget status sentence

### Why it feels like "predefined questions only"

Because that is effectively what it is.

The current `/api/ai/chat` assistant:

- does not interpret user intent deeply
- does not generate answers from a prompt/model call
- only reacts to a small set of hardcoded finance terms
- falls back to a generic finance summary when no branch matches

So it behaves like a keyword-triggered reporting assistant, not a general-purpose AI assistant.

### Grounded context currently stored

Each assistant message may store `grounded_context_json` with:

- `organization_name`
- `total_spend_this_month`
- `pending_approvals`
- `company_budget_remaining`
- `top_department`
- `top_priority`
- `top_priority_reason`
- `overspending`

This is useful if a future refactor wants to preserve auditable grounding.

## Path 2 In Detail: Current Document AI

### Main orchestrator

Defined in `spendpilot-services/app/services/document_service.py`.

Main responsibilities:

- upload documents
- list documents
- read documents from storage
- scan documents
- extract expense data
- store scan results
- update linked expense records

### AI service implementation

Defined in `spendpilot-services/app/services/ai_foundry_service.py`.

This service has two clients:

- `DocumentIntelligenceClient`
- `AzureOpenAI`

### Current extraction flow

#### Text-like files

If the file is text-like:

- content is decoded locally
- extractor is marked `local-text`
- fallback regex extraction may run

#### PDFs and images

If Document Intelligence is configured:

- text extraction uses `prebuilt-read`
- invoice extraction uses `prebuilt-invoice`

If invoice extraction does not provide enough fields:

- OCR text is sent to Azure OpenAI
- model is asked to return compact JSON with:
  - `vendor_name`
  - `invoice_number`
  - `invoice_date`
  - `currency`
  - `total_amount`
  - `category_hint`
  - `summary`

If that also fails:

- fallback regex extraction runs

### Current document analysis prompt behavior

If Foundry is enabled:

- text is sent to Azure OpenAI chat completions
- response format is `json_object`
- system instruction says the model should return compact JSON with:
  - `summary`
  - `risk_level`
  - `findings`
  - `recommendations`
  - optional `extracted_expense`

If Foundry is not enabled or the call fails:

- fallback analysis runs via `policy_service.py`

### Current fallback analysis behavior

Defined in `spendpilot-services/app/services/policy_service.py`.

Fallback analyzer scans for hardcoded terms such as:

- `auto renew`
- `penalty`
- `termination`
- `confidential`
- `indemn`
- `personal data`
- `tax`

It converts those into findings with severities.

If no findings are detected:

- it still creates a low-severity "Human review suggested" finding

This fallback path is also rule-based.

## Current Config And Runtime Wiring

### App config

Defined in `spendpilot-services/app/core/config.py`.

Relevant settings:

- `AZURE_AI_FOUNDRY_ENDPOINT`
- `AZURE_AI_PROJECT_ENDPOINT`
- `AZURE_AI_MODEL_DEPLOYMENT`
- `AZURE_DOCUMENT_INTELLIGENCE_ENDPOINT`

Important computed properties:

- `foundry_openai_base_url`
- `foundry_enabled`
- `document_intelligence_enabled`

### Helm config map

Defined in `spendpilot-helm/charts/spendpilot/templates/configmap.yaml`.

The chart injects:

- `AZURE_AI_FOUNDRY_ENDPOINT`
- `AZURE_AI_PROJECT_ENDPOINT`
- `AZURE_AI_MODEL_DEPLOYMENT`
- `AZURE_DOCUMENT_INTELLIGENCE_ENDPOINT`

### Deployed GitOps values

#### Dev

From `spendpilot-gitops/environments/dev/values/spendpilot-values.yaml`:

- Foundry endpoint: `https://spendpilotnonprodai.cognitiveservices.azure.com/`
- Model deployment: `gpt-4-1-mini`
- Document Intelligence endpoint: `https://spendpilotnonproddoc.cognitiveservices.azure.com/`

This means:

- dev is eligible to use the live Foundry-backed assistant path
- no new Azure secret is required for that path because the app already uses workload identity
- no new RBAC assignment is required if the existing workload identity role assignments remain intact

#### Prod

From `spendpilot-gitops/environments/prod/values/spendpilot-values.yaml`:

- Foundry endpoint: `https://spendpilotprodai2300.cognitiveservices.azure.com/`
- Model deployment: `gpt-4-1-mini`
- Document Intelligence endpoint: `https://spendpilotproddoc2300.cognitiveservices.azure.com/`

This means:

- prod is eligible to use the live Foundry-backed assistant path
- no new Azure secret is required for that path because the app already uses workload identity
- no new RBAC assignment is required if the existing workload identity role assignments remain intact

#### Staging

`spendpilot-gitops/environments/staging/values/spendpilot-values.yaml` currently does not provide a staging workload identity or AI endpoint block.

Operational meaning:

- staging remains fallback-only for the chat assistant today
- this is consistent with staging still being scaffold-only in infra and not yet owning a real runtime identity contract
- once staging becomes a real runtime, the assistant will switch to the Foundry-backed path only after both of these are present:
  - `AZURE_AI_FOUNDRY_ENDPOINT`
  - `AZURE_AI_MODEL_DEPLOYMENT`

## Authentication Model For AI Calls

### Chat assistant

- Uses the normal authenticated app principal
- Scoped by `organization_id`
- Scoped by `user_id`
- Uses app DB data only
- Uses Azure AI Foundry with workload identity when Foundry env vars are configured
- Falls back to the deterministic grounded reply builder when Foundry config is absent or runtime/model/tool execution fails

### Document AI

- Uses Azure identity
- `AzureOpenAI` auth is done with bearer token provider
- token scope: `https://cognitiveservices.azure.com/.default`
- no static OpenAI API key is used in the app path

## Current Strengths

- Chat assistant is tenant-scoped and grounded in real finance data
- Document AI already has a working Azure AI integration path
- JSON outputs are already normalized into typed schemas
- Fallback behavior exists when cloud AI calls fail
- AI chat sessions are persisted in the database

## Current Limitations

### Chat assistant limitations

- Not a real LLM assistant
- Hardcoded keyword routing
- Very small question surface area
- No reasoning over multi-turn context
- No tool calling
- No retrieval layer
- No citations or sources in UI
- No structured output format beyond plain text plus stored context
- Only the first session is effectively used in the frontend flow

### Document AI limitations

- Prompts are compact and basic
- No explicit spend-policy knowledge base or RAG layer
- No confidence scoring contract beyond provider status
- Fallback analyzer is keyword-based
- No advanced cross-document or historical trend reasoning

## Most Important Refactor Insight

If the goal is to improve the "AI assistant" experience, the main work should happen in:

- `spendpilot-services/app/services/ai_chat_service.py`
- `spendpilot-frontend/app/ai-insights/page.tsx`

The current documents AI path is already model-backed.

The current chat assistant path is not.

So if users say:

- "it only answers predefined questions"
- "it feels canned"
- "it is not acting like real AI"

that diagnosis points directly to `AIChatService`, not to Foundry configuration.

## Recommended Refactor Directions

### Option A: Replace chat rules with true LLM grounded answers

Replace `_build_grounded_reply(...)` so it:

1. gathers tenant-scoped finance context
2. serializes a compact structured context payload
3. sends system prompt plus user message plus context to Azure OpenAI
4. returns a natural-language answer plus structured grounding metadata

This is the most direct fix for the current "predefined question" problem.

### Option B: Keep rules as fallback, add LLM as primary

Safer migration path:

1. keep current rule engine
2. add Foundry-backed chat mode
3. fall back to current rules if model call fails

This lowers rollout risk.

### Option C: Add structured finance tools

Instead of passing a big text blob, expose internal finance "tools" conceptually such as:

- get dashboard
- get pending approvals
- get budget summary
- get department breakdown
- get urgent payments
- get recurring payments

Then let the LLM orchestrate tool usage.

This is more scalable if the assistant will become a real product feature.

## Best First Files To Change For A Chat Refactor

If another agent is going to improve the AI assistant itself, the first files to inspect or change are:

1. `spendpilot-services/app/services/ai_chat_service.py`
2. `spendpilot-services/app/api/routes/ai.py`
3. `spendpilot-frontend/app/ai-insights/page.tsx`
4. `spendpilot-services/app/core/config.py`
5. `spendpilot-gitops/environments/dev/values/spendpilot-values.yaml`
6. `spendpilot-gitops/environments/prod/values/spendpilot-values.yaml`

## Safe Constraints For A Refactor

Another agent should preserve these unless intentionally changing them:

- tenant scoping by `organization_id`
- role and data visibility through the authenticated principal
- existing DB session and message persistence
- existing document AI path unless explicitly refactoring it too
- existing Azure identity authentication pattern for Foundry
- existing response envelopes and auth model

## Short Executive Summary

Today, SpendPilot has:

- a fake AI chat assistant for finance insights that is really a keyword-based reporting layer
- a real Azure AI-backed document analysis pipeline for OCR, invoice extraction, and finance document review

If the product goal is to make the assistant actually conversational and capable beyond predefined questions, the primary refactor target is the chat assistant path in `AIChatService`.
