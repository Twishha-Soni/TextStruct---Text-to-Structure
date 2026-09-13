<img width="1328" height="322" alt="textstuct (1)" src="https://github.com/user-attachments/assets/1ec53b16-7169-46a6-8395-f7d111615cc9" />

# TextStruct — Text to Structure

An AI-powered document processing system that ingests unstructured documents (PDFs, DOCX, images), extracts their text (with OCR fallback for scanned/image content), classifies the document type using an LLM, and extracts structured, schema-validated fields specific to that document type — invoices, resumes, purchase orders, application forms, and contracts.

The system is split into a FastAPI backend (extraction, classification, auth, persistence) and a Streamlit frontend (upload, history, and results viewer), containerized independently and orchestrated with Docker Compose.

---

## Features

- **Multi-format ingestion** — PDF, DOCX, PNG, JPG/JPEG, and WEBP uploads.
- **OCR fallback** — scanned PDF pages and embedded images in DOCX files are run through PaddleOCR when no extractable text layer is found.
- **LLM-based classification** — each document is classified into one of five known types (`invoice`, `resume`, `purchase_order`, `application_form`, `contract`) before extraction, with a confidence score.
- **Schema-validated structured extraction** — a dedicated Pydantic schema and prompt per document type, so extraction output is always structured, typed, and validated rather than free-form JSON.
- **JWT-based authentication** — per-user document isolation; every uploaded/extracted document is scoped to the authenticated user.
- **Document history** — users can revisit previously uploaded documents and re-run extraction.
- **Two independent deployment paths** — build-from-source for local development, or pull-prebuilt-image for production.

---

## Architecture

<img width="1314" height="789" alt="Workflow" src="https://github.com/user-attachments/assets/28f9c536-e23b-449d-b38d-254c1a0c8fb6" />


**Request flow:**

1. **Upload** (`POST /uploads`) — file is saved to a temp path, text is extracted via `extract_text()`, which dispatches to a PDF/DOCX/image-specific extractor. Extractors return raw text plus any OCR warnings. The document is stored with `status="uploaded"`.
2. **Extract** (`POST /extract/{document_id}`) — the stored text is passed through a two-stage LLM pipeline in `app/services/llm.py`:
   - **Classification** — a lightweight call (first ~300 chars) against `DocumentTypeClassification`, returning `doc_type` + `confidence`.
   - **Extraction** — the classified type is looked up in a dispatch table mapping `doc_type → (prompt, Pydantic schema)`, and the full document text is extracted into that schema via `with_structured_output()`.
   - The document is updated to `status="extracted"` with `doc_type`, `confidence`, and `extracted_data` (schema dump) persisted.
3. **History / Document detail** — `GET /history` lists a user's documents; `GET /document/{id}` returns full detail including extracted fields, rendered recursively by the Streamlit frontend.

**Why classification is a separate call from extraction:** the extraction schema and prompt used depend on the document type, so the type must be known before the structured-output call is made. This also keeps the classification call cheap (truncated input, small schema) relative to the full extraction call.

---

## Project Structure

```
TextStruct--Text-to-Structure
|
├── docker-compose.local.yml     # Local dev: builds images from source
├── docker-compose.prod.yml      # Production: pulls prebuilt images
├── example.env                  # Template — copy to .env and fill in
│
├── backend/
│   ├── Dockerfile
│   ├── main.py                  # FastAPI app, router registration
│   ├── gemini_model_free_tiert.py   # Lists Gemini models available on your API key's free tier
│   ├── pyproject.toml / uv.lock
│   ├── alembic/                 # DB migrations
│   │   └── versions/
│   └── app/
│       ├── api/                 # Route handlers: upload, extract, history, document, auth
│       ├── auth/                # JWT creation/validation, password hashing
│       ├── database/            # SQLAlchemy models + session
│       ├── schemas/              # Pydantic extraction schemas (one per doc type)
│       ├── prompts/              # LangChain prompt templates (one per doc type + classification)
│       ├── services/
│       │   ├── llm.py            # Classification + extraction orchestration
│       │   ├── extract_text.py   # File-type dispatch for text extraction
│       │   └── file_types_extraction/   # PDF / DOCX / image extractors + OCR
│       
└── frontend/
    ├── Dockerfile
    ├── streamlit_app.py          # Auth screens, upload, history, document viewer
    ├── render_utils.py           # Recursive renderer for arbitrary extracted-field JSON
    └── pyproject.toml / uv.lock
```

---

## Prerequisites

- Docker and Docker Compose
- A Google AI Studio API key (for Gemini) — free tier is sufficient
- Git

---

## Environment Setup

Copy the example env file and fill in your own values:

```bash
cp example.env .env
```

`.env` contents:

```env
MODEL=
GOOGLE_API_KEY=

POSTGRES_USER=
POSTGRES_PASSWORD=
POSTGRES_DB=

DATABASE_URL=postgresql://<username>:<password>@db:5432/<db_name>

JWT_SECRET_KEY=
```

| Variable | Description |
|---|---|
| `MODEL` | Gemini model name to use for classification and extraction (e.g. `gemini-2.5-flash`). See [Checking available models](#checking-available-models-free-tier) below before setting this. |
| `GOOGLE_API_KEY` | Your Gemini API key from [Google AI Studio](https://aistudio.google.com/). |
| `POSTGRES_USER` | Postgres username, used by both the `db` container and `DATABASE_URL`. |
| `POSTGRES_PASSWORD` | Postgres password. |
| `POSTGRES_DB` | Database name to create on first run. |
| `DATABASE_URL` | Full SQLAlchemy connection string. Keep the host as `db` — that's the Compose service name, not `localhost` (containers reach Postgres via the Docker network, not the host loopback). Replace `<username>`, `<password>`, `<db_name>` with the same values as above. |
| `JWT_SECRET_KEY` | Any long random string, used to sign auth tokens. Generate one with `openssl rand -hex 32`. |

### Checking available models (free tier)

Gemini's free tier model availability can change, and not every model is available to every key. Before setting `MODEL` in `.env`, run:

```bash
cd backend
uv run gemini_model_free_tiert.py
```

This prints every model available to your API key. Pick one from the output and set it as `MODEL` in `.env`.

---

## Local Setup (Docker, build from source)

This path builds the backend and frontend images from source on your machine — use this for development and testing changes.

1. Clone the repo:

   ```bash
   git clone https://github.com/Twishha-Soni/TextStruct---Text-to-Structure.git
   cd TextStruct---Text-to-Structure
   ```

2. Set up `.env` as described in [Environment Setup](#environment-setup).

3. In `docker-compose.local.yml`, set the Postgres data volume path to a real directory on your machine:

   ```yaml
   volumes:
     - </path/to/storage/folder>:/var/lib/postgresql/data
   ```

4. Build and start all services:

   ```bash
   docker compose -f docker-compose.local.yml up --build
   ```

   This builds the `backend` and `frontend` images from their respective `Dockerfile`s and starts them alongside a `postgres:16` container.

5. Run database migrations (first run, and after any schema change):

   ```bash
   docker compose -f docker-compose.local.yml exec backend uv run alembic upgrade head
   ```

6. Open the app:

   - Frontend (Streamlit): [http://localhost:8501](http://localhost:8501)
   - Backend (FastAPI docs): [http://localhost:8000/docs](http://localhost:8000/docs)

7. To stop:

   ```bash
   docker compose -f docker-compose.local.yml down
   ```

---

## Production Setup (Docker, prebuilt images)

`docker-compose.prod.yml` does **not** build images — it expects `backend` and `frontend` to already exist as pushed images in a registry, referenced by tag. This keeps the production host lightweight (no build toolchain, no source needed) and makes deploys a matter of pulling a new tag and restarting.

1. Build and tag your images locally (or in CI):

   ```bash
   docker build -t <your-dockerhub-or-ecr-username>/dociq-backend:latest ./backend
   docker build -t <your-dockerhub-or-ecr-username>/dociq-frontend:latest ./frontend
   ```

2. Push them to a registry. Use either:

   - **Docker Hub** — `docker push <your-dockerhub-username>/dociq-backend:latest`
   - **Amazon ECR** — authenticate and push to your ECR repository URI

   Either option works the same way from Compose's perspective — it just needs a pullable image reference.

3. On the production host, fill in the `image:` field for both services in `docker-compose.prod.yml`:

   ```yaml
   backend:
     image: <your-registry>/dociq-backend:latest
   frontend:
     image: <your-registry>/dociq-frontend:latest
   ```

4. Set up `.env` on the host exactly as in [Environment Setup](#environment-setup), and set the Postgres volume path for that host.

5. Pull and start:

   ```bash
   docker compose -f docker-compose.prod.yml pull
   docker compose -f docker-compose.prod.yml up -d
   ```

6. Run migrations against the production database:

   ```bash
   docker compose -f docker-compose.prod.yml exec backend uv run alembic upgrade head
   ```

7. To deploy an update, build/push a new image tag, then on the host:

   ```bash
   docker compose -f docker-compose.prod.yml pull
   docker compose -f docker-compose.prod.yml up -d
   ```

---

## Adding a New Document Type

The extraction pipeline is dispatch-driven, so adding a new document type (e.g. `receipt`) means adding a schema, a prompt, and one dispatch entry — no changes to the extraction/classification flow itself.

1. **Define the schema** — create `backend/app/schemas/receipt_schema.py`, subclassing `ExtractedDocument` from `base_schema.py`:

   ```python
   from pydantic import BaseModel, Field
   from typing import Literal
   from app.schemas.base_schema import ExtractedDocument

   class ReceiptFields(ExtractedDocument):
       document_type: Literal["receipt"] = "receipt"
       merchant_name: str = Field(description="Name of the merchant.")
       total_amount: float = Field(ge=0, description="Total amount paid.")
       # ...additional fields
   ```

2. **Register the type** in `DocType` in `backend/app/schemas/base_schema.py`:

   ```python
   DocType = Literal["invoice", "resume", "purchase_order", "application_form", "contract", "receipt", "unknown"]
   ```

3. **Write the prompt** — create `backend/app/prompts/receipt_prompt.py`, following the existing pattern:

   ```python
   from langchain_core.prompts import ChatPromptTemplate

   receipt_prompt = ChatPromptTemplate.from_messages([
       ("system",
        """You are a data extraction assistant specializing in receipts. Extract the requested fields from the receipt text below. If a field is not present in the document, and the schema allows it to be optional, omit it rather than guessing. Do not fabricate values.IMPORTANT:
   - The field "document_type" MUST be exactly the string: "receipt"
   - Do NOT write "Receipt" or any other variation."""),
       ("human", "Receipt text:\n\n{document_text}")
   ])
   ```

4. **Update the classification prompt** (`backend/app/prompts/classification_prompt.py`) to mention the new type as one the model should consider.

5. **Register in the dispatch table** in `backend/app/services/llm.py`:

   ```python
   from app.prompts.receipt_prompt import receipt_prompt
   from app.schemas.receipt_schema import ReceiptFields

   _DISPATCH = {
       "invoice": (invoice_prompt, InvoiceFields),
       "resume": (resume_prompt, ResumeFields),
       "purchase_order": (purchase_order_prompt, PurchaseOrderFields),
       "application_form": (application_form_prompt, ApplicationFormFields),
       "contract": (contract_prompt, ContractFields),
       "receipt": (receipt_prompt, ReceiptFields),   # new
   }
   ```

That's it — `extract_fields()` classifies against the updated `DocType`, looks up the new entry in `_DISPATCH`, and runs structured extraction against `ReceiptFields` automatically. No frontend changes are needed either, since `render_utils.py` renders extracted fields generically from whatever keys are present in `extracted_data`.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend framework | FastAPI |
| Frontend | Streamlit |
| LLM orchestration | LangChain (`init_chat_model`, structured output) |
| LLM provider | Google Gemini (`google_genai`) |
| Database | PostgreSQL + SQLAlchemy 2.0 |
| Migrations | Alembic |
| Auth | JWT (`python-jose`) + `pwdlib` (Argon2/bcrypt) |
| PDF parsing | PyMuPDF |
| DOCX parsing | python-docx |
| OCR | PaddleOCR + OpenCV |
| Dependency management | `uv` |
| Containerization | Docker, Docker Compose |
