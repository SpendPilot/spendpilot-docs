# Async Email Delivery

## Architecture

SpendPilot now supports asynchronous transactional email with this path:

`backend service -> Azure Service Bus queue -> Azure Function -> Azure Communication Services Email`

Repo locations:

- backend publisher: `spendpilot-services/app/services/email_queue.py`
- finance and onboarding triggers:
  - `spendpilot-services/app/services/finance_service.py`
  - `spendpilot-services/app/services/user_service.py`
- function project: `spendpilot-services/functions/email_sender/`
- Terraform module: `spendpilot-infra/modules/email-delivery/`

## Azure resources

Each environment is now wired to provision:

- one Service Bus namespace
- one `email-requests` queue with duplicate detection enabled
- one Email Communication Service
- one Communication Service associated to the email domain
- one sender username
- one Linux Function App
- one Function storage account
- one Function consumption plan

Function App naming:

- dev: `spendpilot-dev-email-func`
- staging: `spendpilot-staging-email-func`
- prod: `spendpilot-prod-email-func`

## Terraform

Run `plan` and `apply` per environment root:

```powershell
terraform -chdir=spendpilot-infra/envs/dev init
terraform -chdir=spendpilot-infra/envs/dev plan

terraform -chdir=spendpilot-infra/envs/staging init
terraform -chdir=spendpilot-infra/envs/staging plan

terraform -chdir=spendpilot-infra/envs/prod init
terraform -chdir=spendpilot-infra/envs/prod plan
```

## Function deployment

Use the `Email Function Deploy` GitHub Actions workflow:

- push to `main` deploys the function to `dev`
- manual dispatch can deploy to `staging` or `prod`

## Prod custom sender domain

Prod uses a customer-managed sender domain. After the prod Terraform apply, inspect:

- `email_delivery_contract.email_domain_verification_records`

Create the required DNS verification records before expecting prod email sends to succeed.

## End-to-end check

1. Apply the target Terraform environment.
2. Deploy the function code with the GitHub workflow.
3. Trigger a welcome login or finance approval flow.
4. Confirm a message reaches `email-requests`.
5. Confirm the function processes it successfully.
6. Confirm the target mailbox receives the expected template.
