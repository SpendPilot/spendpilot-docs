# Local development context

Recommended local workflow:

1. Copy `.env.example`, `repos/spendpilot-frontend/.env.example`, and `repos/spendpilot-services/.env.example`
2. Keep auth in dev mode:
   - `AUTH_MODE=dev-local`
   - `NEXT_PUBLIC_AUTH_MODE=dev-local`
3. Start the stack with `docker compose up --build`

Useful commands:

```bash
docker compose up --build
curl http://localhost:8000/health
pytest
npm run build
```

Notes:

- Local document uploads store files under `repos/spendpilot-services/data/uploads` when Blob Storage is not configured
- Backend tests use SQLite and the dev auth mode
- Local combined development still runs through `repos/spendpilot-services/app.main:app` even though AKS now builds service-specific images from `repos/spendpilot-services/services/`
