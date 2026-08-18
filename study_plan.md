# One-week study plan — AI College Assistant

Study this repository as **three separate systems** that meet at HTTP:

| System | What it is | Where it lives |
| --- | --- | --- |
| **Backend** | FastAPI HTTP API, auth, storage, orchestration | `app/api/`, `app/services/`, `app/config/`, `app/main.py` |
| **AI / RAG** | Retrieve college documents, then generate answers | `app/ai/`, `app/rag/`, `app/ingestion/` |
| **Flutter** | Student UI that talks to the API | `flutter_app/lib/` |

Do **not** try to learn all three at once. Each day has a primary track. Keep a notebook with: file path, what it does, one question you still have.

**Time:** about **90–120 minutes/day**. If you have less time, do the “Must read” list only.

**How to study a file:** skim the module docstring, then follow one function from input to return. Do not read every line of tests first — use tests to confirm after you think you understand.

---

## Daily checklist (every day)

1. Backend running: `uvicorn app.main:app --reload` → `http://127.0.0.1:8000/docs`
2. Write 3–5 bullet notes: *what I understood / what I still don’t*
3. One small experiment (curl, pytest, or Flutter hot restart)

---

## Day 1 — Big picture (all three, lightly)

**Goal:** Know what the app does and which folder owns which job.

### Must read

- `README.md` — architecture table, API list, source-of-truth policy
- `app/main.py` — how FastAPI is created (CORS, routers, lifespan)
- `app/config/settings.py` — env-driven settings (Gemini, CORS, JWT, RAG knobs)
- `flutter_app/README.md` — how the client is meant to run

### Trace one request (whiteboard, no deep code)

Student types a question in Flutter → `POST /chat/stream` → retrieve chunks → Gemini streams tokens → Flutter paints the answer.

### Hands-on

```powershell
# Backend
cd D:\chatbot
.\.venv\Scripts\Activate.ps1
uvicorn app.main:app --reload

# In another terminal
Invoke-RestMethod http://127.0.0.1:8000/health
```

Open `/docs` and list endpoints you recognize vs ones you don’t.

### Done when

You can explain in one paragraph: **API vs RAG vs Flutter**, and why answers are supposed to cite documents.

---

## Day 2 — Backend only (HTTP, wiring, health)

**Goal:** Treat the backend as a normal web API. Ignore Gemini internals.

### Must read (in this order)

1. `app/main.py` — app factory
2. `app/api/lifespan.py` — startup/shutdown
3. `app/api/routes.py` — `/health`, `/ready`, `/chat`, `/chat/stream`, `/conversations`, `/search`
4. `app/api/schemas.py` — request/response models (the contract Flutter uses)
5. `app/api/dependencies.py` — how `ChatbotService` is built and cached
6. `app/api/middleware.py` — request timing / request id
7. `app/services/readiness.py` — when `/ready` returns 200 vs 503

### Skip today

`app/ai/`, `app/rag/`, `app/ingestion/` (except knowing they exist).

### Hands-on

- Call `GET /health` and `GET /ready` and compare JSON
- Read `tests/test_health.py` and `tests/test_ready.py`
- Run: `pytest tests/test_health.py tests/test_ready.py -q`

### Questions to answer in your notes

- Why is `/health` public but `/chat` protected?
- What does `get_chatbot_service()` do that a route should **not** do itself?
- What must be true for the system to be “chat ready”?

---

## Day 3 — Backend only (auth, rate limit, conversations)

**Goal:** How the API decides who may chat, and how history is stored.

### Must read

1. `app/api/auth.py` — API key **or** JWT
2. `app/api/jwt_auth.py` — create/decode tokens, expiry
3. `app/api/auth_routes.py` — `POST /auth/token`
4. `app/api/rate_limit.py` — per-client limits
5. `app/api/exception_handlers.py` — Gemini errors → HTTP
6. `app/services/memory.py` — in-memory conversation turns
7. `app/services/conversation_factory.py` — json vs sqlite store
8. `app/services/sqlite_conversation_store.py` — persistence
9. `scripts/generate_jwt.py` — local token helper

### Hands-on

```powershell
python scripts/generate_jwt.py --subject student-1
# Then POST /chat with Authorization: Bearer <token>
pytest tests/test_auth_token.py tests/test_ci_auth.py tests/test_sqlite_conversations.py -q
```

### Questions to answer

- When is auth **off** vs **on**? (`API_AUTH_KEY`, `JWT_SECRET`)
- What happens when a JWT expires? (401 `"JWT has expired."`)
- Where is `conversation_id` created and reused?

---

## Day 4 — AI / RAG only (documents → chunks → search)

**Goal:** Understand retrieval **without** generation. This is the “knowledge” half.

### Must read

1. `app/ingestion/models.py` — `SourceDefinition`, `ProcessedDocument`
2. `app/ingestion/parser.py` / `cleaner.py` — HTML/PDF → text
3. `app/ingestion/local_loader.py` — `regulation/` PDFs
4. `app/services/knowledge_base.py` — JSONL store + `rebuild_index`
5. `app/rag/chunking.py` — split documents for embeddings
6. `app/ai/embeddings.py` — `multilingual-e5-base`
7. `app/rag/vector_store.py` — Chroma wrapper
8. `app/rag/query_processing.py` — language / query cleanup
9. `app/rag/retriever.py` — **hybrid** semantic + BM25
10. `sources.yaml` — allowlist of live URLs

### Scripts (read, then optionally run)

- `scripts/seed_knowledge_base.py`
- `scripts/ingest_regulations.py` — local `regulation/arabic_2021.pdf`
- `scripts/ingest_faculty.py` — crawl enabled `sources.yaml` URLs
- `scripts/build_index.py`
- `scripts/evaluate_retrieval.py`

### Hands-on

```powershell
# If index already exists, skip ingest; just inspect
# data/processed/documents.jsonl  and  vector_db/
pytest tests/test_local_loader.py tests/test_evaluation.py -q
```

Open `POST /search` in `/docs` with a regulation question. Note titles and scores.

### Questions to answer

- Why chunk size/overlap exist (`CHUNK_SIZE`, `CHUNK_OVERLAP`)?
- What does `MIN_RETRIEVAL_SCORE` do?
- Why hybrid search (vector + BM25) for course codes and Arabic?

**Mental model:** RAG does **not** fine-tune Gemini on your PDF. It **indexes** the PDF and **pastes relevant excerpts** into the prompt.

---

## Day 5 — AI / RAG only (generation, prompts, streaming)

**Goal:** How retrieved chunks become an answer (and when they don’t).

### Must read

1. `app/ai/llm.py` — `GroundedLLMProvider` interface (Gemini is swappable)
2. `app/ai/prompts.py` — grounded prompt vs general fallback prompt
3. `app/ai/gemini.py` — `generate`, stream, retries on 503
4. `app/ai/gemini_errors.py` — student-facing 429/503/404 messages
5. `app/services/chatbot.py` — **the orchestrator** (read this slowly)
6. `app/api/sse.py` — SSE event format (`status`, `token`, `sources`, `done`, `error`)
7. `CHAT_GENERAL_FALLBACK` in `app/config/settings.py`

### Follow `ChatbotService.answer_stream` on paper

1. Start/resume conversation  
2. Retrieve contexts  
3. If contexts exist → grounded stream + citation markers `[S1]`  
4. If no citations / no contexts → general fallback (if enabled)  
5. Emit `done` with `grounded: true|false`

### Hands-on

```powershell
pytest tests/test_chatbot.py tests/test_gemini.py tests/test_sse.py -q
```

Ask the same question via `/chat` (full JSON) and `/chat/stream` (SSE). Compare.

### Questions to answer

- What is a **citation index** and why uncited answers used to be rejected?
- What is `grounded: false` after the general-fallback change?
- Why SSE errors must be caught **inside** the stream (response already started)?

---

## Day 6 — Flutter only (structure → API client → chat)

**Goal:** The app is a thin client. All RAG lives on the server.

### Must read (in this order)

1. `flutter_app/lib/main.dart` — `CollegeAssistantApp`, locale, theme
2. `flutter_app/lib/config/app_config.dart` — base URL, API key, JWT storage
3. `flutter_app/lib/services/college_api_client.dart` — HTTP + SSE
4. `flutter_app/lib/services/chat_stream_parser.dart` — parse SSE chunks
5. `flutter_app/lib/services/auth_session.dart` — fetch/refresh JWT
6. `flutter_app/lib/models/` — all models (map 1:1 to API JSON)
7. `flutter_app/lib/controllers/chat_controller.dart` — chat state machine
8. Screens: `home_screen.dart` → `chat_screen.dart` → `conversations_screen.dart`
9. Widgets: `message_bubble.dart`, `suggested_questions.dart`

### Hands-on

```powershell
cd D:\chatbot\flutter_app
flutter test
# Then run (use local CanvasKit if gstatic.com is blocked):
flutter run -d chrome --web-port=3000 --no-web-resources-cdn
```

In DevTools Network: watch `POST /auth/token` then `POST /chat/stream`.

### Questions to answer

- Where is `http://127.0.0.1:8000` stored? (`AppConfig`, SharedPreferences)
- Why Flutter web needs CORS origins on the backend
- How `_newId()` must stay web-safe (`nextInt(1 << 32)` is **0** on web)
- How expired JWT is detected and retried

---

## Day 7 — Integration + ops (put the three together)

**Goal:** Run the full loop and know what to change for a real deployment.

### Must read

- `DEPLOY.md` — env vars, CORS, Docker notes
- `docker-compose.yml` / `Dockerfile` if present
- `.github/workflows/` — CI (pytest + Flutter tests)
- `tests/test_flutter_integration.py` — CORS + SSE contract from the **backend** side
- `app/api/admin_routes.py` — ingest/admin (optional)
- `app/services/ocr_queue.py` — scanned PDFs (optional)

### Full-loop exercise (90 min)

1. Start uvicorn  
2. Start Flutter on port 3000 with `--no-web-resources-cdn`  
3. Ask a regulation question → expect **sources**  
4. Ask something not in the KB → expect **general guidance** + `grounded: false`  
5. Wait or fake expiry → confirm JWT refresh  
6. Open Recent conversations  

### Write a one-page “architecture cheat sheet” in your notes

```
Flutter  --JSON/SSE-->  FastAPI routes
                           |
                           v
                     ChatbotService
                      /          \
               Retriever        GeminiProvider
                  |                    |
            Chroma + BM25         prompts.py
                  |
         documents.jsonl  <-  ingest scripts / regulation PDF
```

### Optional stretch

Change one thing and verify:

- Backend: lower `RETRIEVAL_TOP_K` and see `/search` change  
- AI: read `build_grounded_prompt` and explain the anti-injection rules  
- Flutter: add a suggested question in `data/suggested_questions.json` (backend) and see it in the UI  

---

## Track A — Backend (file map)

Study these when you are in “backend mode.” Do not mix with Flutter widgets.

| Topic | Files |
| --- | --- |
| App entry | `app/main.py` |
| Settings | `app/config/settings.py`, `.env.example` |
| Routes & schemas | `app/api/routes.py`, `app/api/schemas.py` |
| DI / caching | `app/api/dependencies.py` |
| Auth | `app/api/auth.py`, `jwt_auth.py`, `auth_routes.py` |
| Rate limit | `app/api/rate_limit.py` |
| SSE wrapper | `app/api/sse.py` |
| Admin | `app/api/admin_routes.py` |
| Chat orchestration* | `app/services/chatbot.py` |
| KB persistence | `app/services/knowledge_base.py` |
| Conversations | `memory.py`, `conversation_factory.py`, `sqlite_conversation_store.py` |
| Readiness | `app/services/readiness.py` |
| Feedback / analytics | `feedback.py`, `analytics.py` |
| Tests | `tests/test_health.py`, `test_ready.py`, `test_auth_token.py`, `test_flutter_integration.py` |

\* `chatbot.py` is backend **orchestration**; the LLM call itself is Track B.

---

## Track B — AI / RAG (file map)

Study these when you are in “AI mode.” This is retrieval + generation, not the Flutter UI.

| Topic | Files |
| --- | --- |
| LLM interface | `app/ai/llm.py` |
| Gemini | `app/ai/gemini.py`, `gemini_errors.py` |
| Prompts | `app/ai/prompts.py` |
| Embeddings | `app/ai/embeddings.py` |
| Chunking | `app/rag/chunking.py` |
| Vector DB | `app/rag/vector_store.py` |
| Query prep | `app/rag/query_processing.py` |
| Hybrid retrieve | `app/rag/retriever.py` |
| Eval | `app/rag/evaluation.py` |
| Ingest models | `app/ingestion/models.py`, `parser.py`, `crawler.py`, `local_loader.py` |
| Allowlist | `sources.yaml` |
| Scripts | `scripts/ingest_*.py`, `seed_knowledge_base.py`, `evaluate_retrieval.py` |
| Data | `data/seed/`, `regulation/`, `data/processed/documents.jsonl`, `vector_db/` |
| Tests | `tests/test_chatbot.py`, `test_gemini.py`, `test_sse.py`, `test_evaluation.py` |

**Concepts to master (not libraries):** embeddings, chunking, hybrid search, grounded prompting, citations, SSE tokens, fallback vs refuse.

This project does **not** use LangChain. The same pipeline is implemented directly: load → chunk → embed → Chroma → retrieve → prompt Gemini.

---

## Track C — Flutter (file map)

Study these when you are in “client mode.” Assume the API already works.

| Topic | Files |
| --- | --- |
| Entry / theme | `lib/main.dart` |
| Config | `lib/config/app_config.dart` |
| HTTP + SSE | `lib/services/college_api_client.dart` |
| SSE parse | `lib/services/chat_stream_parser.dart` |
| Auth | `lib/services/auth_session.dart` |
| Chat state | `lib/controllers/chat_controller.dart` |
| Screens | `lib/screens/home_screen.dart`, `chat_screen.dart`, `conversations_screen.dart` |
| Widgets | `lib/widgets/message_bubble.dart`, `suggested_questions.dart` |
| Models | `lib/models/*.dart` |
| Tests | `flutter_app/test/*.dart` |
| Run notes | `flutter_app/README.md`, `flutter_app/run_web.ps1` |

**Web pitfalls you already hit (re-read when debugging):**

- `Random().nextInt(1 << 32)` → 0 on web  
- CORS + extra headers (`Cache-Control`)  
- JWT stored locally then expired  
- CanvasKit from `gstatic.com` blocked → `--no-web-resources-cdn`  
- `MaterialBanner` requires non-empty `actions`

---

## Suggested order if you get lost

1. `README.md`  
2. `app/api/routes.py` (`/chat` and `/chat/stream` only)  
3. `app/services/chatbot.py` (`answer_stream` only)  
4. `app/rag/retriever.py` + `app/ai/prompts.py`  
5. `flutter_app/lib/services/college_api_client.dart` (`streamChat`)  
6. `flutter_app/lib/controllers/chat_controller.dart` (`sendMessage`)

That path is the whole product.

---

## What *not* to study in week 1

- All of `.venv/` and generated Flutter `build/`  
- Every Chroma/sentence-transformers internals  
- Rewriting the stack in LangChain  
- Production Kubernetes unless you finish Day 7 early  

---

## End-of-week self-test

You should be able to answer **without opening the repo**:

1. **Backend:** Which route streams chat, and which dependency builds `ChatbotService`?  
2. **AI:** What happens if retrieval returns nothing?  
3. **Flutter:** Which class turns SSE events into on-screen tokens?  
4. **Ops:** Why Flutter on `:3000` talking to API on `:8000` needs CORS?  
5. **Safety:** Why college rules should still prefer cited chunks over general Gemini knowledge?

If you can answer those five, you have studied the project — not just scrolled it.
