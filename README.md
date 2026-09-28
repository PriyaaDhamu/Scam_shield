# ScamShield

> An explainable, student-focused scam-risk analysis prototype for suspicious offers, messages, documents, and links.

**Repository note:** This repository contains source code and synthetic development data. Local databases, caches, Python bytecode, Node dependencies, and compiled frontend output are intentionally excluded from Git.


**Don't trust. Verify.** A student-focused, explainable scam-risk analysis prototype for suspicious internship and job offers, scholarship notices, payment requests, phishing links, messages, screenshots, and PDFs.

ScamShield combines independent rule-based evidence, passive URL structure checks, sender/domain consistency checks, and a TF-IDF + Logistic Regression language model. It reports High Risk, Medium Risk, Low Risk, or Unable to Determine. A result is guidance, never a guarantee or an accusation.

> **Demo and model limitations:** All included examples and training rows are fictional synthetic development data. The held-out metrics are calculated on that synthetic corpus and must not be represented as production or real-world accuracy. External URL reputation is not configured. ScamShield never visits submitted URLs.

## Architecture and structure

See [ARCHITECTURE.md](ARCHITECTURE.md) for the data flow, trust boundaries, and complete file map. FastAPI provides validated analysis, upload, demo, dashboard, and model metadata endpoints. The React/TypeScript client renders live results and dashboard data. SQLite is the local default; PostgreSQL can be selected with `DATABASE_URL`.

## Requirements

- Python 3.11+ recommended (tested here with Python 3.14)
- Node.js 20.19+ or 22.12+
- For screenshot OCR: the Python `pytesseract` package **and the separate Tesseract OCR executable** on `PATH`. Without the executable, the upload API returns a clear 503 error; text/PDF and all other features continue to work.
- For PostgreSQL: a reachable PostgreSQL server and a configured `DATABASE_URL`; local SQLite requires no service.

## Local development

From the repository root, create and activate a virtual environment, install the backend dependencies, and start the API:

```powershell
py -3 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r backend\requirements.txt
Set-Location backend
python -m uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

In a second terminal, install and start the frontend:

```powershell
Set-Location frontend
npm install
npm run dev
```

Open <http://localhost:5173>. The API is at <http://localhost:8000>; interactive OpenAPI docs are at <http://localhost:8000/docs>. To point the frontend at a different API origin, define `VITE_API_URL` (for example `http://localhost:8000/api`) before starting Vite.

To use the VS Code Live Server extension, run `npm run build` from `frontend/` first, then open `frontend/index.html` with Live Server (default port 5500). The source page forwards to `frontend/dist/index.html`, which contains the compiled app. For custom Live Server ports, open the `dist/index.html` file directly and include that origin in `CORS_ORIGINS`.

Do not expose this prototype publicly without adding authentication, authorization, rate limits, CSRF/origin protections appropriate to deployment, upload malware controls, operational logging/retention policy, and a privacy/security review. CORS is not authentication. No accounts or multi-user access are implemented.

## Risk engine

The editable rule weights and thresholds are in `backend/app/config.py`, with environment-backed thresholds. Rule detectors independently report fee/payment asks, credentials/OTP, urgency, threats, reward claims, impersonation/authority language, verification pressure, sensitive documents, secrecy, sender-domain mismatches, and forceful writing. URL features are inspected locally. Individual points are capped per URL evidence group.

The score is the sum of distinct evidence points and a capped synthetic-model contribution, bounded from 0 to 100. High/medium thresholds are configurable. Short messages use `Unable to Determine`. Lack of HTTPS is reported as a caution only and contributes no points. No single signal constitutes proof.

## Machine learning

`backend/app/services/classifier.py` reads the clearly labeled `backend/app/data/synthetic_training.csv`, reserves a stratified 25% test split using a fixed random seed, reports precision, recall, F1, and a 2x2 confusion matrix, then fits the inference pipeline on the full synthetic corpus. Run from `backend/`:

```powershell
python train_model.py
```

The live `/api/model-info` endpoint exposes the computed values and dataset caveat. They are not benchmark claims and the corpus is not a substitute for consented, representative, independently labeled data. The classifier is modular so a later model can replace it without changing evidence schemas.

## Upload and URL behavior

- Text: up to 20,000 characters.
- Upload: PDF, PNG, JPEG, or WebP; configurable 8 MB default. PDFs are limited to 20 pages and selectable text is extracted locally with `pypdf`. Scanned PDFs with no text request an image/screenshot instead.
- Images: local OCR via Tesseract. OCR may be unavailable or fail; the API returns explicit errors rather than inventing extracted text.
- URLs: checks HTTPS state, raw IP hosts, nested subdomains, punycode, long addresses, certain unusual characters, URL shorteners, and selected host naming patterns. These are explainable heuristics, not reputation verdicts. The application does not fetch DNS, follow redirects, or open links. Email/link domain comparisons are heuristic and cannot verify ownership.
- Stored history: SQLite/PostgreSQL stores category, source, risk level/score, evidence count, and evidence keys for aggregates. Raw messages, uploaded files, and extracted text are not saved. Fictional demo analyses are not added to history.

## API overview

All request bodies and responses are documented by FastAPI at `/docs` and `/openapi.json`.

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/api/health` | Service health |
| `POST` | `/api/analyze` | Analyze JSON `{ "text": "...", "category": "Internship" }` |
| `POST` | `/api/analyze/upload` | Multipart file upload with `category` and optional `context` |
| `GET` | `/api/url-check?url=...` | Passive structural inspection; never opens URL |
| `GET` | `/api/demos` | List explicitly fictional examples |
| `POST` | `/api/demos/{demo_id}/analyze` | Run an example through analysis without persisting it |
| `GET` | `/api/dashboard` | Aggregate history (no raw message text) |
| `GET` | `/api/model-info` | Dataset metadata and synthetic-only held-out metrics |

The result includes a risk label/score, factors, quoted evidence snippets, URL checks, recommended next steps, uncertainty notes, and synthetic-model caveat.

## Tests and production build

```powershell
Set-Location backend
python -m pytest tests -q
Set-Location ..\frontend
npm run build
```

## Three-minute demo flow

1. Open the Overview and point out the empty/real dashboard and local model caveat.
2. Select **Demo center** and run **Internship Scam** (all cards are marked fictional).
3. Walk through the independent fee, identity, secrecy, contact, and URL evidence.
4. Expand **Why did ScamShield say this?** and explain point contributions and synthetic-model limits.
5. Return to the Overview to show the persisted aggregate record; demo cases themselves are not persisted.
6. Inspect a link with **URL inspection**; show the passive/no-network behavior.
7. Under **Student scam check**, upload an image after installing Tesseract, or a selectable-text PDF. Without the optional OCR executable, screenshot upload clearly reports that OCR is unavailable.

## Deployment notes

Build the client with `npm run build`; serve `frontend/dist` behind a production web server. Run the API with an ASGI server such as `uvicorn app.main:app --host 0.0.0.0 --port 8000` from `backend/`, using a process manager/container orchestrator and externally managed PostgreSQL in production. Set `CORS_ORIGINS` to the exact frontend origin, set `DATABASE_URL`, use TLS, and provide health checks against `/api/health`. Add authentication and abuse controls before sharing a deployment. Keep external reputation integration disabled unless an approved provider is configured, documented, and handled under an explicit data policy.
