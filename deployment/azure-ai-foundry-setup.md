# Azure AI Foundry and Document Intelligence

Terraform provisions:

- Azure AI Foundry/OpenAI account (`AIServices`)
- Azure OpenAI model deployment
- Azure AI Document Intelligence account (`FormRecognizer`)
- Blob storage container for uploaded files

Application behavior:

1. upload invoice or receipt
2. store file in Blob Storage
3. extract OCR text with Document Intelligence
4. extract invoice fields with the prebuilt invoice model
5. send text and extracted fields to Azure AI Foundry
6. return summary, risk level, findings, recommendations, and structured expense hints

Fallback behavior remains in code so local development still works without Azure AI configured.
