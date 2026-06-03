# API Drift Detection — CI

## What it is

GitHub Actions workflow that runs on every push to `main` in `YM_frontend`.

Verifies that the manually written frontend types (`contracts.ts`) have not drifted from the backend schemas — without changing a single line of frontend code.

---

## Why it exists

The frontend holds manually written types in `features/*/api/contracts.ts` that mirror the backend Pydantic schemas.

There is no code generation — types are written by hand and kept accurate through discipline. The CI guards against future drift.

---

## How it works

```
push to main
     ↓
checkout YM_Backend (public repo)
     ↓
export_openapi.py → openapi.json
(APP_ENV=test, no DB, no lifespan)
     ↓
openapi-typescript → generated.ts
     ↓
git diff --exit-code generated.ts
     ↓
✅ matches baseline → pass
❌ differs → fail + instructions
```

---

## Key files

| File | Role |
|------|------|
| `frontend/.github/workflows/api-drift.yml` | the workflow |
| `frontend/src/types/generated.ts` | committed baseline — 14k lines, reference only |
| `backend/scripts/tooling/export_openapi.py` | generates openapi.json from the app object without a running server |
| `backend/app/main.py` | `/openapi.json` disabled in production |

---

## When the CI fails

The backend changed a schema that was not updated in the frontend. Steps to fix:

```bash
# 1. Generate openapi.json from local backend
cd backend
APP_ENV=test JWT_SECRET=x ./.venv/bin/python -m scripts.tooling.export_openapi --output ../openapi.json

# 2. Regenerate generated.ts
cd ..
npx openapi-typescript openapi.json -o frontend/src/types/generated.ts

# 3. Review what changed
git diff frontend/src/types/generated.ts

# 4. Update the relevant contracts.ts accordingly

# 5. Commit the updated generated.ts together with the contracts change
```

---

## What generated.ts is not

`generated.ts` is a **smoke detector only** — it is never imported in application code.

The actual API layer remains in `features/*/api/contracts.ts` with clean ergonomics.

---

## Security

`/openapi.json` is disabled in production (`APP_ENV == "production"`) in `backend/app/main.py`.

The CI generates the schema locally inside the job — it does not pull from production.
