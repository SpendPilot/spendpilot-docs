# VM deployment

Use the assets in `infra/vm/`.

High-level steps:

1. Provision a VM and assign managed identity.
2. Install Docker and Nginx.
3. Copy the compose and Nginx files.
4. Set environment variables.
5. Start the stack.

Recommended external dependencies:

- PostgreSQL
- Azure Blob Storage if file storage is centralized
- Azure AI Foundry and Document Intelligence
