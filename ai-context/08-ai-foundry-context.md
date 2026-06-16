# Azure AI context

Primary AI flow:

1. upload finance document
2. store the file in Blob Storage
3. extract OCR text with Document Intelligence
4. extract invoice fields with the prebuilt invoice model
5. analyze the text and extracted fields with Azure AI Foundry
6. return finance-oriented summary, risks, findings, recommendations, and expense hints

Important files:

- `repos/spendpilot-services/app/services/ai_foundry_service.py`
- `repos/spendpilot-services/app/services/document_service.py`
- `repos/spendpilot-services/app/services/policy_service.py`

Fallback behavior:

- local regex extraction for invoice hints
- keyword-based risk detection if Azure AI is unavailable

Implementation note:

- extracted expense payloads can contain decimal totals, so document metadata and scan raw responses must be JSON-encoded safely before writing to PostgreSQL JSON columns
