# AI College Assistant

A production-oriented Retrieval-Augmented Generation (RAG) backend for answering Mansoura University Faculty of Engineering questions from approved, traceable sources.

## Current status: Phase 20 OCR queue and conversation list

Phases 1–19 are complete. Phase 20 adds an OCR review queue for scanned PDFs, `GET /conversations` for recent chat summaries, `GET /admin/ocr-queue`, and a Flutter **Recent conversations** screen.

### Architecture decisions

| Concern | Choice | Why |
| --- | --- | --- |
| API | FastAPI | Async-friendly, typed OpenAPI contract, and simple Flutter integration. |
| Generation | `GeminiProvider` interface, targeting `gemini-3.7-flash` | Keeps Gemini replaceable and supports fast multilingual responses. |
| Embeddings | `intfloat/multilingual-e5-base` | Local, efficient multilingual retrieval model; supports Arabic and English in one vector space. |
| Initial vector store | ChromaDB | Persistent local storage with metadata filtering and a simple Python API; wrap it behind `VectorStore` so Qdrant/Pinecone/Weaviate can replace it later. |
| Retrieval | Hybrid (semantic + BM25) in Phase 4 | Better resilience for regulation numbers, course codes, and Arabic wording. |

### Source-of-truth policy

College-specific answers must be grounded in retrieved, approved sources. When retrieval confidence is below the configured threshold, the service must say that it could not find reliable information. Generation never supplies regulations, deadlines, fees, or procedures without a citation.

### Known risks to address in later phases

- Official websites/PDFs can be stale or conflicting; preserve URL, publication date, fetch date, and document version metadata.
- Arabic PDFs may have poor text extraction; add OCR only for scanned documents and flag it in metadata.
- Prompt injection inside documents must be treated as content, not instructions.
- Retrieval scores are not calibrated confidence; evaluate them against a labeled Arabic/English question set before setting a production threshold.
- Respect `robots.txt`, official policies, rate limits, and a source allowlist during ingestion.

## Run locally

Requires Python 3.11+.

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements-dev.txt
Copy-Item .env.example .env
uvicorn app.main:app --reload
```

Open `http://127.0.0.1:8000/docs`, or verify the service:

```powershell
Invoke-RestMethod http://127.0.0.1:8000/health
pytest
```

## Current API contract

- `GET /health` — application status.
- `GET /ready` — deployment readiness for chat (returns `503` until KB, index, and Gemini are configured).
- `POST /chat` — grounded answer, source citations, confidence, and conversation ID.
- `POST /chat/stream` — same answer via Server-Sent Events for incremental Flutter rendering.
- `GET /conversations` — recent server-side conversation summaries.
- `GET /conversations/{conversation_id}` — bounded server-side conversation history.
- `GET /suggested-questions?language=en|ar` — starter prompts for an empty chat screen.
- `POST /feedback` — anonymous helpful/not-helpful signal after an answer.
- `POST /search` — retrieval-only results for debugging or a future search UI.
- `GET /sources` — configured source registry, including enabled state.

`/chat`, `/chat/stream`, `/search`, and `/feedback` return `503` with a clear setup message until documents have been ingested and indexed; `/chat` and `/chat/stream` also require `GEMINI_API_KEY`. No endpoint silently falls back to ungrounded answers.

When `API_AUTH_KEY` is set, protected routes require an `X-API-Key` header. When `JWT_SECRET` is set, protected routes also accept `Authorization: Bearer <token>`. `/health`, `/sources`, and `/suggested-questions` remain public.

Issue a JWT from the API (requires `JWT_SECRET`; uses `X-API-Key` when `API_AUTH_KEY` is configured):

```powershell
Invoke-RestMethod -Method POST `
  -Uri http://127.0.0.1:8000/auth/token `
  -Headers @{ "X-API-Key" = "your_shared_key" } `
  -ContentType "application/json" `
  -Body '{"subject":"student-1"}'
```

Or generate a development token locally:

```powershell
python scripts/generate_jwt.py --subject student-1
```

In the Flutter app, open **Settings → Get JWT from API** after entering your API key.

Chat endpoints enforce `RATE_LIMIT_PER_MINUTE` (default `30`) per client IP. Set `RATE_LIMIT_PER_MINUTE=0` to disable limiting during local testing.

### Admin API (`ADMIN_API_KEY` required)

| Route | Purpose |
| --- | --- |
| `GET /admin/documents` | List processed knowledge-base documents. |
| `POST /admin/documents` | Add one reviewed document without crawling. |
| `DELETE /admin/documents/{document_id}` | Remove a document from the knowledge base. |
| `POST /admin/reindex` | Rebuild the Chroma vector index from processed documents. |
| `POST /admin/ingest/{source_id}` | Fetch one enabled allowlist source, upsert, and reindex. |
| `POST /admin/ingest-all` | Fetch every enabled source, upsert, and reindex. |
| `GET /admin/sources` | List allowlist sources with last-ingest status. |
| `GET /admin/sources/check` | Verify robots.txt and URL reachability before enabling a source. |
| `GET /admin/ocr-queue` | Scanned PDFs flagged for manual OCR review. |
| `GET /admin/analytics` | Anonymous aggregate metrics (questions, feedback, latency). |

Send the configured key in the `X-Admin-Key` header. Admin routes return `503` until `ADMIN_API_KEY` is set.

### SSE event contract (`POST /chat/stream`)

| Event | Purpose |
| --- | --- |
| `status` | Retrieval/generation stage updates (`retrieving`, `generating`). |
| `token` | Incremental answer text for the Flutter chat bubble. |
| `sources` | Final citations and confidence before completion. |
| `error` | Safe user-facing message when generation fails (e.g. Gemini 503). |
| `done` | Final payload matching `/chat` (`conversation_id`, `answer`, `sources`, `confidence`, `grounded`). |

Configure Flutter web origins through `CORS_ORIGINS` (comma-separated).

## Roadmap

1. Foundation and environment setup (complete)
2. Approved-source ingestion and document processing (complete)
3. Embeddings and Chroma persistence (complete)
4. Hybrid retrieval and citations (complete)
5. Gemini provider integration (complete)
6. Chat service and lightweight memory (complete)
7. Complete API integration (complete)
8. Evaluation, tests, and performance work (complete)
9. Flutter-ready streaming, feedback, and auth integration points (complete)
10. Seed knowledge base, persistence, admin API, and analytics (complete)
11. Flutter client scaffold (`flutter_app/`) (complete)
12. Docker deployment and live ingestion pipeline (complete)
13. GitHub Actions CI, JWT auth, and chat rate limiting (complete)
14. Live faculty source verification and bulk ingestion workflow (complete)
15. Flutter web connectivity and interactive home screen fixes (complete)
16. JWT token issuance endpoint and Flutter bearer-token support (complete)
17. Retrieval evaluation, ingestion history, and Flutter conversation history (complete)
18. Production readiness probe, Docker healthcheck, and CI retrieval gate (complete)
19. SQLite conversation persistence with JSON migration (complete)
20. OCR review queue and Flutter conversation list (complete)

## Enable live faculty ingestion

Reviewed Mansoura faculty URLs are listed in [`sources.yaml`](sources.yaml) but remain **disabled** until you verify them locally:

```powershell
python scripts/ingest_faculty.py --check
python scripts/ingest_faculty.py --check --source faculty-home --timeout 45
```

When a source shows `ALLOWED`, set `enabled: true` for that entry in `sources.yaml`, then ingest and evaluate:

```powershell
python scripts/ingest_faculty.py --ingest --source faculty-home --evaluate --include-live-eval
python scripts/ingest_faculty.py --ingest --evaluate --include-live-eval
```

Or via the admin API (requires `ADMIN_API_KEY`):

```powershell
Invoke-RestMethod -Headers @{ "X-Admin-Key" = "your_admin_key" } `
  http://127.0.0.1:8000/admin/sources

Invoke-RestMethod -Headers @{ "X-Admin-Key" = "your_admin_key" } `
  http://127.0.0.1:8000/admin/sources/check

Invoke-RestMethod -Method POST -Headers @{ "X-Admin-Key" = "your_admin_key" } `
  http://127.0.0.1:8000/admin/ingest-all
```

Ingestion attempts are logged under `data/runtime/ingestion_history.jsonl`.

If checks time out, retry with a higher `--timeout` value or run from a network that can reach `engfac.mans.edu.eg`.

## Docker deployment

See [`DEPLOY.md`](DEPLOY.md). Quick start:

```powershell
docker compose build
docker compose up -d
docker compose run --rm api python scripts/update_knowledge_base.py --seed
```

Before enabling live faculty sources:

```powershell
python scripts/check_sources.py
```

## Flutter client

See [`flutter_app/README.md`](flutter_app/README.md). Quick start:

```powershell
cd flutter_app
flutter create . --project-name college_assistant --platforms=android,ios,web
flutter pub get
flutter run -d chrome --web-port=3000
```

## Quick start with seed data

For local development without crawling the live faculty website, load the tracked seed documents and build the vector index:

```powershell
.\.venv\Scripts\Activate.ps1
python scripts/update_knowledge_base.py --seed
```

This writes approved Arabic/English sample regulations into `data/processed/documents.jsonl` and rebuilds `vector_db/`. After that, `/health` should report `knowledge_base_ready: true` once indexing completes.

Runtime persistence (conversations, feedback, analytics) is stored under `data/runtime/` and is excluded from version control. Set `CONVERSATION_STORE=sqlite` in `.env` to use SQLite instead of `conversations.json` (existing JSON is migrated automatically on first startup).

## Ingest approved sources

Review [sources.yaml](sources.yaml) before ingestion. A source must be explicitly enabled, and the crawler checks `robots.txt` before each source. It fetches one URL at a time, waits at least two seconds between requests, accepts only HTML/PDF content, and enforces a 10 MB document limit. Re-ingesting the same page replaces the existing document instead of duplicating it.

```powershell
python scripts/check_sources.py
python scripts/ingest.py --list
python scripts/ingest.py --source faculty-home
python scripts/update_knowledge_base.py --source faculty-home
```

Raw files are stored in `data/raw/<source-id>/`; normalized records are appended to `data/processed/documents.jsonl`. Do not treat an ingestion run as approval of every discovered link: add each new starting URL to `sources.yaml` after review.

## Build the local vector index

After ingestion has created `data/processed/documents.jsonl`, build a local persistent Chroma index. The first run downloads the `intfloat/multilingual-e5-base` model. It uses the E5-required `passage: ` prefix for source chunks and `query: ` for questions.

```powershell
python scripts/build_index.py --dry-run
python scripts/build_index.py
```

The index is stored under `vector_db/` and is intentionally excluded from version control. Re-running the command is idempotent: Chroma upserts chunks with stable IDs.

## Configure Gemini

Create a local `.env` file from `.env.example`, then add a newly created Gemini API key. Never commit it or paste it into chat.

```text
GEMINI_API_KEY=your_replacement_key
GEMINI_MODEL=gemini-3.7-flash
```

`GeminiProvider` uses low-temperature generation and emits only an answer grounded in supplied retrieval context. It requires source markers such as `[S1]`; markers are parsed into structured citation IDs for the chat service.

## Evaluate retrieval quality

After indexing reviewed sources, run the versioned Arabic, Egyptian Arabic, and English evaluation set:

```powershell
python scripts/evaluate_retrieval.py --verbose
python scripts/ingest_and_evaluate.py --seed --verbose
python scripts/ingest_faculty.py --check --timeout 45
python scripts/ingest_faculty.py --ingest --source faculty-home --evaluate --include-live-eval
```

It reports Recall@K and MRR against expected source IDs. Use `--min-recall 0.8` in CI to fail when quality drops. Expand `data/evaluation_queries.json` for seed content and `data/evaluation_queries_live.json` after enabling live faculty sources.
