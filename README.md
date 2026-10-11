# Financial Report Analyzer

A production-style **RAG** application that reads company annual reports (10-K
PDFs), extracts financial KPIs into a comparison dashboard, and answers
follow-up questions via an AI analyst chatbot.

It runs entirely on a **free, open stack** (Google Gemini + local embeddings +
pgvector), with a clean path to swap in Azure services for a cloud deployment.

| Concern        | This project (free)                       | Azure equivalent         |
| -------------- | ----------------------------------------- | ------------------------ |
| LLM            | Google Gemini (free tier)                 | Azure OpenAI             |
| Embeddings     | **fastembed** `bge-small` (local, no API) | Azure OpenAI embeddings  |
| Vector store   | **pgvector** (inside Postgres)            | Azure AI Search          |
| Structured DB  | PostgreSQL                                 | Azure PostgreSQL         |
| UI             | Streamlit                                 | —                        |
| Backend        | FastAPI                                   | FastAPI                  |
| Local run      | Docker Compose                            | —                        |
| Deploy (opt.)  | Kubernetes manifests (`k8s/`)             | ACR + AKS                |

### Why local embeddings?

Ingesting a full 100+ page 10-K produces hundreds of chunks. Gemini's free-tier
embedding endpoint caps at ~100 requests/minute with a low daily quota, which
can't ingest a real filing. So the **high-volume** work (embeddings) runs
locally with `fastembed` (ONNX, no torch, no quota), while **low-volume** LLM
calls (one KPI extraction per document, plus chat) use Gemini. Set
`EMBEDDING_PROVIDER=gemini` to use the API instead (only practical for small docs).

Gemini's free chat tier is also tight (~5 requests/minute). Uploads therefore
**store chunks first** (always succeeds) and extraction is a separate step: if it
is rate-limited, the document is still searchable and you click **Generate KPIs**
(one call) to fill them in.

## How it works

```
PDF ─▶ Markdown (pymupdf4llm) ─▶ chunk ─▶ embed ─▶ pgvector
                                                      │
                          per-KPI retrieval query ◀───┘
                                   │
                                   ▼
                     Gemini → structured JSON KPIs ─▶ Postgres ─▶ dashboard
                                   │
                     chat question → retrieve → Gemini → answer
```

KPIs are extracted **once** at ingestion and served from Postgres (cheap reads);
the chatbot queries the vector store live. This is a deliberate cost-saving design.

## Module layout

```
app.py                 FastAPI entry (startup runs DB init)
config.py              all settings (env-driven; free ⇄ Azure)
ingestion/             pdf_to_markdown, chunker, ingest_documents
embeddings/provider    local fastembed (default) / Gemini API
llm/gemini             structured JSON + chat completions
vectorstore/pgvector   store + hybrid retriever (vector + keyword)
rag/kpi_extractor      retrieve context → LLM → FinancialMetrics
database/              db init, save_metrics, metrics (read)
routes/                health, ingestion, chat, dashboard
frontend/streamlit_app Streamlit UI (Dashboard / Upload / AI Analyst)
k8s/                   phase-2 Kubernetes manifests
```

## Quick start (Docker Compose)

1. Get a free Gemini API key: https://aistudio.google.com/app/apikey
2. Configure env:
   ```bash
   cp .env.example .env
   # edit .env and set GEMINI_API_KEY
   ```
3. Launch:
   ```bash
   docker compose up --build
   ```
4. Open:
   - UI:      http://localhost:8501
   - API docs: http://localhost:8000/docs
5. Add reports: see [scripts/download_sample_reports.md](scripts/download_sample_reports.md),
   then upload in the **Upload** tab (name files `2025_Apple.pdf`).

## Run without Docker (dev)

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
# Start a local Postgres+pgvector (or use the compose `db` service) and set
# POSTGRES_HOST=localhost in .env, then:
uvicorn app:app --reload
# In another shell:
pip install -r frontend/requirements.txt
BACKEND_URL=http://localhost:8000 streamlit run frontend/streamlit_app.py
```

## Tests & CI

```bash
pip install ruff pytest && ruff check . && pytest
```

GitHub Actions (`.github/workflows/ci.yml`) lints, tests, and builds both images
on every push.

## Switching to Azure (phase 2)

The provider seams are already in place. Adding an `azure` branch to
`llm/`, `embeddings/`, and `vectorstore/` plus wiring the `k8s/` manifests to
ACR/AKS gives a cloud deployment without touching the app logic.

---

*Credits: the architecture is inspired by a public Azure-based RAG tutorial by
[AaiTech](https://github.com/Sandesh-hase/AI-Powered-Investor-Intelligence-Platform);

<img width="1822" height="1185" alt="image" src="https://github.com/user-attachments/assets/ca569e50-5dda-4b32-9752-839b6fbc5cae" />

this is an independent, open-stack implementation.*
