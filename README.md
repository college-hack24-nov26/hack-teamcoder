# KYC Verification API

For the short judging walkthrough, local setup steps, and the boundary between synthetic samples and live checks, see [HACKATHON_DEMO.md](HACKATHON_DEMO.md).

KYC as a service. Other platforms call this API with an API key — no Docker, Postgres, or Redis required.

## Run (Windows / macOS / Linux)

You need **Node.js 20+**.

```sh
npm run setup
npm start
```

The API listens at **http://localhost:8000**.

`npm run setup` installs dependencies, creates a local SQLite database, and prints:

- Admin login: `approver@verifai.com` / `admin`
- Demo platform API key: `kyc_live_local_dev_do_not_use_in_prod`

## Integrate from another platform

Send `X-API-Key: kyc_live_...` on every request.

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/v1` | API catalog |
| `POST` | `/v1/kyc/sessions` | Create a KYC session (`externalUserId` required) |
| `PUT` | `/v1/kyc/sessions/:sessionId/applicant` | Save name, DOB, address, IDs |
| `POST` | `/v1/kyc/sessions/:sessionId/documents` | Upload `file` + `documentType` (`PASSPORT`, `AADHAR`, `BIOMETRIC`, …) |
| `POST` | `/v1/kyc/sessions/:sessionId/submit` | Start verification |
| `GET` | `/v1/kyc/sessions/:sessionId` | Poll status / results |
| `PUT` | `/v1/webhooks` | Set `webhookUrl` for `kyc.session.updated` |

Example:

```sh
curl -X POST http://localhost:8000/v1/kyc/sessions \
  -H "X-API-Key: kyc_live_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"externalUserId\":\"user-123\",\"applicant\":{\"firstName\":\"Ada\",\"lastName\":\"Lovelace\",\"dateOfBirth\":\"1815-12-10\"}}"
```

Issue more keys after logging in as admin:

```sh
POST /v1/api-keys
Authorization: Bearer <jwt>
{ "name": "payments-app" }
```

## Python document service (port 8001)

Use **Python 3.11** (already installed as `py -3.11`). Do not use the default 3.14 interpreter.

```sh
py -3.11 -m venv document-service/.venv
document-service\.venv\Scripts\python -m pip install -r document-service/requirements-local.txt
document-service\.venv\Scripts\python document-service/manage.py migrate
document-service\.venv\Scripts\python document-service/manage.py seed_schemas
npm run dev:ml
```

Health: http://localhost:8001/health

This service receives `/process-kyc` from the Node API and posts results to `/api/kyc/ml-callback`. The local requirements install EasyOCR for document text recognition and TensorFlow CPU 2.18 plus DeepFace for face matching (NumPy 1.26.4 keeps the ML stack compatible). PDF scans also require Poppler installed on the host. If the ML stack or provider credentials are unavailable, the service sends the application to manual review; it never approves based only on uploaded files.

## App login APIs (existing dashboard)

- `POST /api/auth/register`
- `POST /api/auth/login`
- `POST /api/kyc/application` (JWT)

Health check: `GET /health`
