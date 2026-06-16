# Troubleshooting

Backend ready check shows database error:

- Verify `DATABASE_URL`
- Confirm network access from the runtime to PostgreSQL

Frontend sign-in fails:

- Check `NEXT_PUBLIC_ENTRA_FRONTEND_CLIENT_ID`
- Check `NEXT_PUBLIC_ENTRA_API_SCOPE`
- Check whether the app authority is `https://login.microsoftonline.com/common`
- Confirm `NEXT_PUBLIC_ENTRA_API_SCOPE` uses the backend Application ID URI, not the backend client ID
- Confirm frontend API helpers are not generating `/api/api/...` paths when `NEXT_PUBLIC_API_BASE_URL` is already `/api`
- Verify redirect URIs in the Entra app registration

Login redirects back to the same page and shows raw HTML in red:

- The frontend likely received an HTML fallback page instead of a JSON API response
- Check the browser network tab for `/api/auth/me`
- If the request path is `/api/api/auth/me`, the frontend API base path is being prefixed twice
- The shared `buildApiUrl` helper in this repo now normalizes duplicate `/api` prefixes and converts HTML error responses into readable messages

Azure sign-in page says "You can't sign in here with a personal account":

- The Entra app registration is still restricted to work or school accounts
- Or the frontend authority is still `organizations` instead of `common`

External tenant sign-in fails with `AADSTS650052`:

- The customer tenant does not have the frontend and backend enterprise applications provisioned yet
- Confirm the backend API registration bundles consent through known client applications and pre-authorized frontend access
- Ask a tenant admin in that customer tenant to open the admin-consent URL and complete consent if user consent is blocked or if service-principal creation has not happened yet

Backend token validation fails:

- Check `ENTRA_BACKEND_AUDIENCE`
- Check `ENTRA_AUTHORITY`
- Confirm the frontend requested the backend scope
- The backend accepts the configured Application ID URI plus both client-ID audience forms (`<client-id>` and `api://<client-id>`) to support Microsoft identity platform v2 tokens across personal and organizational accounts
- Users do not need to be pre-registered in the app; after Entra consent, a first-time sign-in should bootstrap the user automatically
- Browser sign-in consent for the frontend app does not automatically grant Azure CLI consent to call the same API; CLI-based token tests may require a separate `az login --scope <api-scope>` flow

Document scan produces fallback results:

- Verify Azure Document Intelligence endpoint
- Verify Azure AI Foundry endpoint and deployment
- Confirm managed identity has access

Document scan or expense extraction returns `500 Internal Server Error` after OCR succeeds:

- Check the documents-service logs for JSON serialization errors around extracted invoice totals
- This repo now stores AI extraction payloads with JSON-safe encoding so `Decimal` invoice totals do not break PostgreSQL JSON columns
- If the failure returns after a deploy, confirm the latest backend image rolled out successfully in AKS

Guest Microsoft account creates a different workspace than a native user in the same Entra tenant:

- Guest accounts such as `name_example.com#EXT#@tenant.onmicrosoft.com` should join the tenant workspace, not a separate personal workspace
- If they are being split out, verify the backend is running the latest auth bootstrap code that only isolates true consumer-tenant tokens
