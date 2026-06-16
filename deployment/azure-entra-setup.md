# Azure Entra Setup

Terraform now creates the core app registrations:

1. frontend SPA app registration
2. backend API app registration

Backend API registration:

- supported account types: `AzureADandPersonalMicrosoftAccount`
- application ID URI: `api://<prefix>-<env>-api`
- scope: `access_as_user`
- known client applications: include the frontend SPA app ID so customer-tenant consent can provision both enterprise apps together
- pre-authorized applications: include the frontend SPA app with the `access_as_user` permission scope
- app roles:
  - `platform_admin`
  - `org_owner`
  - `dept_head`
  - `employee`
  - legacy aliases such as `org_admin`, `finance_manager`, `approver`, and `auditor` may still appear in older environments, but the product now canonicalizes memberships to `org_owner`, `dept_head`, and `employee`

Frontend SPA registration:

- supported account types: `AzureADandPersonalMicrosoftAccount`
- authority: `https://login.microsoftonline.com/common`
- requested access token version: `2`
- request the API scope as `<backend Application ID URI>/access_as_user`
- register delegated API access to the backend `access_as_user` scope
- local redirect URI for `http://localhost:3000/login`
- Front Door login redirect URI based on the deployed endpoint hostname

Customer tenant onboarding does not require the user to be pre-created inside the app database.

- The first successful browser sign-in bootstraps the user automatically.
- If the customer's Entra tenant allows user consent, a normal user can complete the first sign-in flow.
- If the customer's Entra tenant blocks user consent, a tenant admin must grant consent to the Enterprise Application first.
- External-tenant enterprise applications and service principals are created by consent in that tenant. Your Terraform in the home tenant cannot directly pre-create them in every customer tenant.
- The Terraform output `entra_admin_consent_url_template` gives the tenant-admin consent URL pattern for external customer tenants.
- Tenant-side app role assignment is optional in this repo because the app can also manage tenant roles internally after bootstrap.

Personal Microsoft accounts are also supported.

- A true Microsoft consumer-tenant sign-in gets one isolated workspace per personal account so consumer users do not collapse into the shared Microsoft consumer tenant.
- A guest Microsoft account invited into a workforce tenant stays in that workforce tenant workspace and follows that tenant's role model.

The app uses delegated user access, not a broad application identity for end-user finance actions:

- users sign in with Microsoft and obtain the backend `access_as_user` scope
- the backend maps the token to a tenant-scoped organization membership
- all finance and document queries are filtered by `organization_id`
- role checks such as `org_owner`, `dept_head`, and `employee` then decide which resources and actions are allowed within that tenant
