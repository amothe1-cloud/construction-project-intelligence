# BuildDocs AI — Project Overview

A complete reference for the **construction-project-intelligence** repo. This document describes what the project is, how it is organized, how the ingestion and chat pipelines work, and the operational details (auth, RBAC, deployment, schema, env vars).

> The product is also called **BuildDocs AI**. Live URLs from the [README.md](README.md): app at `constructionrag.vercel.app`, API at `builddocs-api.vercel.app`.

---

## 1. What it is

A **role-aware construction document intelligence platform** ("RAG for construction"). Users:

1. Create a project (e.g., a real construction job).
2. Add team members with one of five roles, optionally scoped to a trade.
3. Upload construction documents — drawings, specifications, RFIs, submittals, daily reports, meeting minutes, safety docs, change orders, or general PDFs/Word/Excel/CSV/images.
4. Documents are processed (text extraction → chunking → optional embedding → indexed).
5. Users **chat** with the project. Every answer is grounded in the project's own documents and returns citations with file name, page number, snippet, and a relevance score.
6. **RBAC enforces visibility** at every layer (list, view, search, retrieval), so a subcontractor never sees an owner-only document — even indirectly via chat.

Two LLM providers are first-class: **Groq** (`llama-3.1-8b-instant`, free tier) and **Anthropic Claude**. An OpenAI-compatible endpoint can be plugged in. If no API key is configured, a built-in **TF-IDF extractive Q&A** fallback returns the most relevant sentences from the documents — no LLM required.

---

## 2. Repository layout

This is a **pnpm + Python monorepo** (pnpm workspace declared in [pnpm-workspace.yaml](pnpm-workspace.yaml)).

```
construction-project-intelligence/
├── apps/
│   ├── api/      FastAPI backend (Python) — primary serverless API on Vercel
│   ├── web/      Next.js 15 frontend (TypeScript)
│   └── worker/   Legacy Celery worker (used by the docker-compose dev stack)
├── packages/
│   └── shared/
│       ├── python/   Shared Python role definitions (mirrors apps/api/app/shared_roles.py)
│       └── src/      Shared TypeScript role / visibility / permission enums
├── infra/
│   ├── docker/   Dockerfiles for api/web/worker (used by docker-compose)
│   └── scripts/  seed.py demo data, import_datasets.py
├── e2e/          Playwright end-to-end tests (admin-flow, subcontractor-restriction)
├── docker-compose.yml   Full local stack: postgres, redis, qdrant, minio, api, worker, web
├── render.yaml          Render.com deploy config (alternative to Vercel)
├── .env.example         Required env vars
├── pnpm-workspace.yaml  pnpm workspace definition
└── README.md            User-facing readme
```

**Two parallel deployment paths** coexist intentionally:

| Path | Backend runtime | Storage | DB | Vector store | Worker | Status |
|------|----------------|---------|----|--------------|--------|--------|
| **Production (Vercel)** | FastAPI on Vercel Python serverless | Supabase Storage | Supabase Postgres (PostgREST) | None / pgvector via RPC | None — FastAPI `BackgroundTasks` or inline ingestion | **Active** |
| **Local docker-compose** | FastAPI in container | MinIO | Postgres 16 + Alembic | Qdrant | Celery + Redis | Legacy / dev |

The API code (`apps/api`) is structured to run in **either** mode, branching on the `VERCEL` env var. The Celery worker in `apps/worker` is the older path and is no longer needed for the Vercel deploy — its logic was largely ported into [apps/api/app/services/ingestion.py](apps/api/app/services/ingestion.py).

---

## 3. Tech stack

### Frontend ([apps/web](apps/web))
- **Next.js 15** (App Router) + **React 19** + TypeScript
- **Tailwind CSS** + **shadcn/ui** (Radix UI primitives)
- **TanStack Query v5** for server-state caching
- **React Hook Form** + **Zod** for forms
- **react-pdf** + **pdfjs-dist** for in-browser PDF viewing
- **react-dropzone** for drag-and-drop upload
- **lucide-react**, `date-fns`, `clsx`, `tailwind-merge`, `class-variance-authority`

The web app proxies `/api/*` to the Python backend via Next.js rewrites — see [apps/web/next.config.ts](apps/web/next.config.ts). The default upstream is `https://builddocs-api.vercel.app`, overridable via `API_BACKEND_URL`.

### Backend ([apps/api](apps/api))
- **FastAPI** (async) — entry [apps/api/app/main.py](apps/api/app/main.py), Vercel handler [apps/api/api/index.py](apps/api/api/index.py)
- **Pydantic v2** + **pydantic-settings** for config — see [apps/api/app/config.py](apps/api/app/config.py)
- **Supabase Async client** (PostgREST) — wired up in [apps/api/app/supabase_client.py](apps/api/app/supabase_client.py) and [apps/api/app/deps.py](apps/api/app/deps.py)
- **JWT** via `python-jose`, password hashing via `passlib[bcrypt]` — see [apps/api/app/security/jwt.py](apps/api/app/security/jwt.py)
- **PDF parsing**: `pypdf` on Vercel (lightweight ~2MB), `pymupdf`/`fitz` locally (page image rendering)
- **DOCX**: `python-docx`; **XLSX**: `openpyxl`; **CSV**: stdlib `csv`
- **Embeddings (local)**: `fastembed` (ONNX, CPU, ~100 MB) — `BAAI/bge-small-en-v1.5`, dim 384
- **Reranker (local)**: `fastembed` `TextCrossEncoder` — `cross-encoder/ms-marco-MiniLM-L-6-v2`, with a keyword-overlap fallback
- **HTTP**: `httpx`

### Requirements split

| File | Purpose |
|------|---------|
| [apps/api/requirements.txt](apps/api/requirements.txt) | Lightweight serverless deps (~15 MB bundle). FastAPI, supabase, pypdf, python-docx, openpyxl, jose, passlib, httpx. |
| [apps/api/requirements-local.txt](apps/api/requirements-local.txt) | Heavy ML extras for local dev: `pymupdf`, `fastembed`, `uvicorn[standard]`, `pytest`. |
| [apps/api/requirements-vercel.txt](apps/api/requirements-vercel.txt) / `requirements-full.txt` | Variants for build experimentation. |

### AI / RAG
- **LLM providers**: Groq (default), Anthropic Claude, OpenAI-compatible — see [apps/api/app/ai/provider.py](apps/api/app/ai/provider.py)
- **Generator**: [apps/api/app/ai/generator.py](apps/api/app/ai/generator.py) — picks provider, falls back to extractive Q&A
- **Embedding**: [apps/api/app/ai/embeddings.py](apps/api/app/ai/embeddings.py)
- **Reranker**: [apps/api/app/ai/reranker.py](apps/api/app/ai/reranker.py)
- **Vision (VLM)**: [apps/api/app/ai/vlm.py](apps/api/app/ai/vlm.py) — used by the legacy worker for page image analysis (Anthropic vision or OpenAI vision)

### Deployment
- **Vercel** — frontend (Next.js) and Python serverless API (separate Vercel projects). Vercel config at [apps/api/vercel.json](apps/api/vercel.json).
- **Supabase** — managed Postgres + Storage. The API talks to Postgres via PostgREST exclusively (no SQLAlchemy in the production hot path).
- **Render.com** — alternative backend deployment via [render.yaml](render.yaml).
- **docker-compose** ([docker-compose.yml](docker-compose.yml)) — full local stack with postgres, redis, qdrant, minio, api, worker, web.

---

## 4. Architecture

### High-level (Vercel production)

```
                  ┌──────────────────────────────┐
   Browser ──────►│ Next.js 15 (Vercel)          │
                  │  apps/web — App Router        │
                  │  TanStack Query, shadcn/ui    │
                  └──────────────┬────────────────┘
                                 │  rewrite /api/* → API
                                 ▼
                  ┌──────────────────────────────┐
                  │ FastAPI on Vercel Python      │
                  │  apps/api/api/index.py        │
                  │  ─ auth (JWT cookies)         │
                  │  ─ projects / members         │
                  │  ─ documents (upload/list)    │
                  │  ─ chat (RAG)                 │
                  │  ─ audit                      │
                  └──┬─────────────┬──────────────┘
                     │             │
                     ▼             ▼
        ┌────────────────┐   ┌─────────────────────┐
        │ Supabase       │   │ LLM provider        │
        │  ─ Postgres    │   │  Groq / Anthropic / │
        │  ─ Storage     │   │  OpenAI-compatible  │
        │  ─ pgvector    │   │  (or extractive QA) │
        └────────────────┘   └─────────────────────┘
```

### Local stack (docker-compose)

The compose file additionally provisions Redis, Qdrant, MinIO, and a Celery worker. Document ingestion runs as a Celery task ([apps/worker/worker/tasks/ingest.py](apps/worker/worker/tasks/ingest.py)) instead of a FastAPI BackgroundTask. Vector search uses Qdrant; storage uses MinIO; the API uses SQLAlchemy/Alembic against Postgres directly.

### Module map (`apps/api/app`)

```
app/
├── main.py              FastAPI app + CORS (allows *.vercel.app + localhost)
├── config.py            Pydantic settings (env-driven)
├── supabase_client.py   Async Supabase client factory
├── deps.py              FastAPI dependencies: get_sb, get_current_user, get_project_membership
├── shared_roles.py      Roles, scopes, doc types, status enums + RBAC maps
├── api/                 Route handlers (one router per resource)
│   ├── router.py        Aggregator
│   ├── auth.py          /auth/{register,login,logout,me}  (cookie JWT)
│   ├── projects.py      /projects (CRUD + member counts + my_role)
│   ├── members.py       /projects/{id}/members
│   ├── documents.py     /projects/{id}/documents (upload, list, detail, download, page image, reprocess)
│   ├── chat.py          /projects/{id}/chat/sessions[/{sid}/messages]
│   └── audit.py         /projects/{id}/audit
├── schemas/             Pydantic request/response models (auth, project, member, document, chat, audit)
├── services/
│   ├── storage.py       Local FS or Supabase Storage backends
│   ├── ingestion.py     Document pipeline (full + lightweight serverless variant)
│   ├── chat_service.py  RAG orchestration (search + rerank + generate + persist)
│   └── audit_service.py Single insert helper for audit_logs
├── ai/
│   ├── provider.py      OpenAICompatibleProvider, AnthropicProvider
│   ├── embeddings.py    fastembed local or API
│   ├── reranker.py      fastembed cross-encoder + keyword fallback
│   ├── generator.py     Provider routing + SYSTEM_PROMPT + extractive Q&A fallback
│   └── vlm.py           Vision model wrapper (legacy worker uses this)
├── retrieval/
│   ├── hybrid_search.py Embedding → keyword fallback → rerank
│   └── qdrant_store.py  pgvector search via Supabase RPC `search_chunks`
├── rbac/
│   └── filters.py       get_allowed_scopes(membership) → list[str]
├── security/
│   └── jwt.py           access (12h) + refresh (7d) tokens, bcrypt hashing
└── models/              SQLAlchemy ORM models — used by the legacy Alembic / Celery path only
```

The web app structure ([apps/web/src/app](apps/web/src/app)) uses Next.js route groups:

```
src/app/
├── (auth)/
│   ├── layout.tsx
│   ├── login/page.tsx
│   └── register/page.tsx
├── (dashboard)/
│   ├── layout.tsx
│   ├── dashboard/page.tsx
│   └── projects/[projectId]/
│       ├── page.tsx        Project home
│       ├── documents/      List + [docId] detail + upload
│       ├── chat/           RAG chat UI
│       ├── members/        Member management
│       └── audit/          Audit log viewer
├── layout.tsx
└── page.tsx              Landing page

src/lib/
├── api-client.ts        Typed fetch wrapper (credentials: "include", 401 → /login)
├── auth-context.tsx     React context for current user
├── query-provider.tsx   TanStack Query provider
└── utils.ts
```

---

## 5. Database schema

Defined in the Alembic migration [apps/api/alembic/versions/001_initial_schema.py](apps/api/alembic/versions/001_initial_schema.py). The Vercel deploy talks to Supabase Postgres via PostgREST, so tables/columns must match this schema in the Supabase project.

| Table | Purpose | Notable columns |
|-------|---------|-----------------|
| `users` | Account record | `email` (unique), `password_hash`, `is_active`, `is_superadmin` |
| `projects` | A construction project | `code` (unique short id), `name`, `location`, `description` |
| `project_memberships` | User ↔ project link with a role | `role`, `assigned_trade` (for subcontractors); unique on (`user_id`,`project_id`) |
| `documents` | Uploaded file metadata | `storage_key`, `file_type`, `doc_type`, `visibility_scope`, `trade_scope`, `revision`, `status`, `page_count`, `processing_error`, `metadata_json` |
| `document_pages` | One row per logical page | `raw_text`, `cleaned_text`, `page_summary`, `image_storage_key`, `extracted_json` (VLM output) |
| `document_chunks` | Retrieval unit | `chunk_index`, `chunk_text`, `metadata_json` (`chunk_type`), `visibility_scope`, `trade_scope`, `vector_id` |
| `chat_sessions` | Per user, per project conversation | `title` (auto-derived from first message) |
| `chat_messages` | One row per turn | `role` ('user'|'assistant'), `content`, `citations_json`, `model_metadata_json` |
| `audit_logs` | Append-only event log | `action` (e.g. `document.upload`, `chat.query`), `entity_type`, `entity_id`, `details_json` |

A Supabase **RPC function `search_chunks`** is expected for the embedding path. It's called from [apps/api/app/retrieval/qdrant_store.py](apps/api/app/retrieval/qdrant_store.py) and must accept `p_project_id`, `p_embedding`, `p_allowed_scopes`, `p_trade_scope`, `p_limit` and return rows with a `similarity` column. (When `VERCEL=1` the API skips this and uses the keyword fallback exclusively.)

---

## 6. Auth

Cookie-based JWT auth — see [apps/api/app/api/auth.py](apps/api/app/api/auth.py) and [apps/api/app/security/jwt.py](apps/api/app/security/jwt.py).

- **Register / login** issue an access token (12 h) and refresh token (7 d), both set as `httponly` cookies.
- `samesite=lax`; `secure=true` only when running on Vercel/Render (detected via env var).
- The web app's `apiFetch` always sends `credentials: "include"`. On 401, it redirects to `/login` (except for `/auth/*` calls).
- Bearer tokens in `Authorization` headers are accepted as a fallback.
- `is_superadmin` users automatically have admin role in every project (synthesized in `get_project_membership`).

---

## 7. RBAC — roles, scopes, permissions

Defined twice (kept in sync intentionally): [apps/api/app/shared_roles.py](apps/api/app/shared_roles.py) for Python, and [packages/shared/src/roles.ts](packages/shared/src/roles.ts) / [visibility.ts](packages/shared/src/visibility.ts) / [permissions.ts](packages/shared/src/permissions.ts) for the frontend.

### Roles

| Role | What they do |
|------|--------------|
| `admin` | Full access; manage members and visibility |
| `project_manager` | Same as admin in practice, except cannot create projects |
| `superintendent` | Field lead — can upload daily reports / safety / general; sees most docs except management-only |
| `subcontractor` | Trade-scoped — can upload submittals/RFIs/general; sees only field_team + own trade_scoped |
| `owner_viewer` | Read-only; only sees `owner_shared` docs |

### Visibility scopes

`project_full`, `field_team`, `trade_scoped`, `owner_shared`, `management_only`. Documents are tagged with one scope at upload. The `ROLE_VISIBILITY_MAP` maps each role to the scopes it may see.

### How RBAC is enforced

Every read query is filtered:

```python
allowed_scopes = get_allowed_scopes(membership)        # rbac/filters.py
sb.table("documents").select("*").in_("visibility_scope", allowed_scopes)
```

For subcontractors, `trade_scoped` documents are further filtered to those whose `trade_scope` matches the user's `assigned_trade`. This is applied:
- on document list ([apps/api/app/api/documents.py:54-61](apps/api/app/api/documents.py#L54-L61))
- on document detail (404 if not allowed)
- in keyword fallback retrieval ([apps/api/app/retrieval/hybrid_search.py](apps/api/app/retrieval/hybrid_search.py))
- in pgvector search (passed to the `search_chunks` RPC)

Upload allowlists per role (`ROLE_UPLOAD_DOC_TYPES`) are enforced in the upload handler. Member management and audit log viewing are gated by `ROLE_PERMISSIONS`.

The chat retrieval guarantee: **if no accessible chunk matches a query, the system explicitly answers "I don't have enough accessible documentation…"** — never leaks even an excerpt of a restricted doc.

---

## 8. Document ingestion pipeline

Two implementations:

### A. Vercel serverless path (active) — [apps/api/app/services/ingestion.py](apps/api/app/services/ingestion.py)

Triggered inline on `POST /api/projects/{id}/documents/upload` because Vercel BackgroundTasks die when the response returns. The function `ingest_document_lightweight()` is called synchronously and is engineered to fit within the **10 s serverless timeout**:

1. **Upload** the original file to Supabase Storage at `projects/{project_id}/documents/{doc_id}/original.{ext}`.
2. **Insert** the `documents` row with `status='processing'`.
3. **Download** the file back from storage (the storage client is the source of truth for the worker).
4. **Extract text** by file type:
   - PDF → `pypdf.PdfReader` (no page image rendering — saves time and memory). Falls back to `pymupdf`/`fitz` if `pypdf` is missing.
   - DOCX → `python-docx` paragraphs + table cells, virtual pages of ~3000 chars.
   - XLSX → `openpyxl` read-only mode, sheet rows joined with ` | `, 50-row pages.
   - CSV → stdlib reader, 50-row pages.
   - PNG/JPG → upload as the only page; no OCR in serverless.
5. **Insert** `document_pages` rows (batched 50 at a time).
6. **Chunk** pages with type-aware logic in `_chunk_pages()`:
   - `drawing` → one chunk per page (entire page text).
   - `specification` → split on numbered section headers `\d+\.\d+`.
   - `rfi` / `submittal` → one record-level chunk per document (capped at 3000 chars).
   - `daily_report` → paragraph-based, max 800 chars.
   - everything else → paragraph-based, 800 chars + 100-char overlap.
7. **Insert** `document_chunks` rows in batches of 50 with `metadata_json={"chunk_type": …}`, `visibility_scope`, `trade_scope`, and a `vector_id` (= chunk id, unused unless embeddings are written).
8. **Skip embeddings** when `VERCEL=1` — chunks live in Postgres and are searched by keyword. (Locally, `_get_embeddings()` runs `fastembed` and updates rows via the Supabase `update_chunk_embedding` RPC.)
9. Mark `status='ready'`. On any exception, mark `status='error'` with `processing_error` set to the (truncated) message.

If a document gets stuck in `chunking` (e.g. earlier deploy crashed mid-way), `POST /api/projects/{id}/documents/{did}/reprocess` re-reads existing pages, deletes old chunks, re-chunks, and stores fresh chunks ([apps/api/app/api/documents.py:255-333](apps/api/app/api/documents.py#L255-L333)).

### B. Local Celery worker — [apps/worker/worker/tasks/ingest.py](apps/worker/worker/tasks/ingest.py)

Runs in docker-compose. Adds:
- Page **image rendering** at 150 DPI (saved to MinIO).
- Optional **VLM page analysis** — sends page image + OCR text to Anthropic vision or OpenAI vision; expects JSON `{page_type, title, summary, metadata, entities, table_data}` and stores it in `document_pages.extracted_json`.
- Embedding via `fastembed` and **upsert into Qdrant** (collection per project; named vector `dense`).
- Status transitions: `processing → rendering_pages → chunking → embedding → ready`.

This path is mostly cosmetic for the Vercel deploy but is preserved for self-hosted deployments and as documentation of the "ideal" pipeline.

---

## 9. Chat / RAG pipeline

Entry point: `POST /api/projects/{id}/chat/sessions/{sid}/messages` → [apps/api/app/api/chat.py](apps/api/app/api/chat.py) → `process_chat_message()` in [apps/api/app/services/chat_service.py](apps/api/app/services/chat_service.py).

```
user message
   │
   ├─► insert user row in chat_messages
   │
   ├─► resolve allowed_scopes from membership.role
   │   resolve trade_scope (only for subcontractor)
   │
   ├─► hybrid_retrieve()  ┌─ embed query  (skipped on Vercel)
   │   (initial=20,        ├─ pgvector RPC search_chunks  (if embedding succeeded)
   │    final_top_k=5)     ├─ if 0 results → keyword fallback
   │                       │      • split query, drop stopwords, keep words >2 chars
   │                       │      • take top 3 longest terms
   │                       │      • ILIKE %term% across document_chunks scoped by RBAC
   │                       │      • if still 0, ILIKE on document_pages.raw_text
   │                       │      • score = (# query words present) / (# query words)
   │                       └─ rerank candidates
   │                              • local cross-encoder if available
   │                              • else keyword-overlap rerank with position bonus
   │                              • return top final_top_k
   │
   ├─► load file_name for each retrieved chunk's document
   │
   ├─► load last 6 messages of chat history for the session
   │
   ├─► generate_answer(query, context_chunks, history)
   │      • if API key configured → LLM call with SYSTEM_PROMPT
   │      • else → extractive Q&A (TF-IDF over candidate sentences,
   │              groups winners by source, prepends [Source N] markers)
   │
   ├─► insert assistant row with citations_json
   │
   ├─► if session.title == "New Chat" → set title to first 80 chars of query
   │
   └─► audit_logs.insert(action="chat.query")
```

The `SYSTEM_PROMPT` (in [apps/api/app/ai/generator.py:62-78](apps/api/app/ai/generator.py#L62-L78)) is strict: no outside knowledge, cite every claim with `[Source N]`, prefer latest revisions, summarize conflicts, refuse to answer when evidence is insufficient.

Confidence is reported as the mean of the top chunks' rerank scores. Each `CitationItem` carries `document_id`, `file_name`, `page_number`, a 200-char `snippet`, and `relevance_score`.

---

## 10. API surface

All routes are mounted under `/api`. Full list:

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/health` | none | Liveness probe (returns `{"status":"ok"}`) |
| POST | `/api/auth/register` | none | Create account; sets cookies |
| POST | `/api/auth/login` | none | Cookie login |
| POST | `/api/auth/logout` | none | Clear cookies |
| GET | `/api/auth/me` | cookie | Current user |
| GET | `/api/projects` | cookie | Projects the user is a member of (or all, if superadmin) |
| POST | `/api/projects` | cookie | Create project; creator becomes admin |
| GET | `/api/projects/{id}` | cookie | Project detail with counts |
| GET | `/api/projects/{id}/members` | membership | List members |
| POST | `/api/projects/{id}/members` | `member.manage` | Add member by email |
| PATCH | `/api/projects/{id}/members/{mid}` | `member.manage` | Update role / trade |
| GET | `/api/projects/{id}/documents` | membership | RBAC-filtered list with optional filters (`doc_type`, `visibility_scope`, `trade_scope`, `status`, `search`) |
| POST | `/api/projects/{id}/documents/upload` | `document.upload` | Multipart upload; triggers ingestion |
| GET | `/api/projects/{id}/documents/{did}` | RBAC | Document detail + pages |
| GET | `/api/projects/{id}/documents/{did}/download` | RBAC | Download original |
| GET | `/api/projects/{id}/documents/{did}/pages/{n}/image` | RBAC | Rendered page PNG (only if image was rendered, i.e. local path) |
| POST | `/api/projects/{id}/documents/{did}/reprocess` | membership | Rebuild chunks from existing pages |
| GET | `/api/projects/{id}/chat/sessions` | membership | List user's sessions |
| POST | `/api/projects/{id}/chat/sessions` | membership | Create session |
| GET | `/api/projects/{id}/chat/sessions/{sid}` | session-owner | Session messages |
| POST | `/api/projects/{id}/chat/sessions/{sid}/messages` | session-owner | Send message → RAG answer |
| GET | `/api/projects/{id}/audit` | `audit.view` | Recent audit log entries (action filter, paginated up to 200) |

Allowed upload extensions: `.pdf .png .jpg .jpeg .docx .xlsx .csv` ([apps/api/app/api/documents.py:17](apps/api/app/api/documents.py#L17)).

CORS in [apps/api/app/main.py](apps/api/app/main.py) accepts:
- explicit `BACKEND_CORS_ORIGINS`
- localhost:3000
- regex `https://.*\.vercel\.app`

---

## 11. Environment variables

Full list in [.env.example](.env.example). The most important ones:

### Required for the API
| Var | Description |
|-----|-------------|
| `SECRET_KEY` | JWT signing key (random 64-char string) |
| `SUPABASE_URL` | Supabase project URL |
| `SUPABASE_SERVICE_KEY` | Service role key (bypasses RLS — used server-side only) |
| `SUPABASE_STORAGE_BUCKET` | Default `documents` |
| `STORAGE_BACKEND` | `supabase` (prod) or `local` (filesystem under `./storage/`) |
| `BACKEND_CORS_ORIGINS` | JSON list of additional origins to allow |

### LLM
| Var | Default | Notes |
|-----|---------|-------|
| `LLM_PROVIDER` | `groq` | `groq` \| `anthropic` \| `openai` |
| `GROQ_API_KEY` | — | https://console.groq.com (free) |
| `GROQ_MODEL` | `llama-3.1-8b-instant` | |
| `ANTHROPIC_API_KEY` | — | |
| `ANTHROPIC_MODEL` | `claude-sonnet-4-20250514` | |
| `ANTHROPIC_VISION_MODEL` | `claude-sonnet-4-20250514` | Used by VLM in worker |
| `LLM_BASE_URL`, `LLM_API_KEY`, `LLM_MODEL` | localhost defaults | OpenAI-compatible fallback |

### Embeddings / reranker
| Var | Default | Notes |
|-----|---------|-------|
| `EMBEDDING_PROVIDER` | `local` | `local` (fastembed) or `api` |
| `EMBEDDING_MODEL` | `BAAI/bge-small-en-v1.5` | |
| `EMBEDDING_DIM` | `384` | |
| `RERANKER_PROVIDER` | `local` | |
| `RERANKER_MODEL` | `cross-encoder/ms-marco-MiniLM-L-6-v2` | |

### Token lifetimes
| Var | Default |
|-----|---------|
| `ACCESS_TOKEN_EXPIRE_MINUTES` | 720 (12 h) |
| `REFRESH_TOKEN_EXPIRE_MINUTES` | 10080 (7 d) |

### Frontend
| Var | Notes |
|-----|-------|
| `NEXT_PUBLIC_API_URL` | Direct API URL when not relying on Next rewrites |
| `API_BACKEND_URL` | Server-side rewrite target in [apps/web/next.config.ts](apps/web/next.config.ts) |

### Magic env vars
| Var | Effect in code |
|-----|----------------|
| `VERCEL=1` | Set automatically by [apps/api/api/index.py](apps/api/api/index.py). Code checks this to: skip embedding (ingestion + retrieval), skip the local reranker, run ingestion inline instead of as a BackgroundTask. |
| `RENDER` | Treated like Vercel for cookie-secure detection. |

---

## 12. Local development

### Option A: API + web only (matches Vercel layout)
```bash
cp .env.example .env
# Fill in SUPABASE_URL, SUPABASE_SERVICE_KEY, SECRET_KEY, optionally GROQ_API_KEY

# API
cd apps/api
pip install -r requirements.txt -r requirements-local.txt
uvicorn app.main:app --reload --port 8000

# Web (in another terminal)
cd apps/web
npm install   # or: pnpm install from repo root
npm run dev
```

Open `http://localhost:3000`. API docs at `http://localhost:8000/docs`.

### Option B: full docker-compose stack
```bash
docker compose up
```

Brings up Postgres, Redis, Qdrant, MinIO (with `documents` bucket auto-created), API (with `alembic upgrade head` on boot), Celery worker, and Next.js dev server.

### Seed data — [infra/scripts/seed.py](infra/scripts/seed.py)
```bash
DATABASE_URL=postgresql+psycopg://... python infra/scripts/seed.py
```
Creates a `Riverside Commercial Complex` project and 5 demo accounts (password `builddocs123`):
- `admin@builddocs.ai` — superadmin
- `sarah.pm@example.com` — project_manager
- `mike.super@example.com` — superintendent
- `jose.electrical@example.com` — subcontractor (trade=`electrical`)
- `owner@riverside.com` — owner_viewer

### Tests
- Backend pytest suite at [apps/api/tests](apps/api/tests): `test_rbac.py`, `test_permissions_filter.py`, `test_chat_retrieval.py`, `test_upload_states.py`.
- E2E with Playwright at [e2e/tests](e2e/tests): `admin-flow.spec.ts`, `subcontractor-restriction.spec.ts`. Configured by [e2e/playwright.config.ts](e2e/playwright.config.ts).

---

## 13. Operational notes & gotchas

- **Vercel 10-second timeout** dominates ingestion design. The `_lightweight` path skips page-image rendering and embeddings; chunks rely on keyword search (`ILIKE %term%`) until the document is reprocessed off-Vercel or until pgvector is wired up via the `search_chunks` Supabase RPC.
- **No SQL endpoint via PostgREST for vectors.** Updating `vector` columns goes through the RPC `update_chunk_embedding` ([apps/api/app/services/ingestion.py:389-411](apps/api/app/services/ingestion.py#L389-L411)). That RPC must exist in your Supabase project.
- **Keyword retrieval is intentionally generous.** It tries the top-3 longest terms across `document_chunks`, then falls back to `document_pages.raw_text` if zero chunks exist (handles documents stuck in `chunking`).
- **Subcontractor trade filter is applied in Python** because PostgREST cannot easily express `(visibility_scope != 'trade_scoped') OR (trade_scope = X) OR (trade_scope IS NULL)`.
- **Cookie-based auth requires `credentials: "include"`** — the api-client wrapper already does this. CORS must allow credentials, which is why a regex (`https://.*\.vercel\.app`) is used instead of `allow_origins=["*"]`.
- **Symlink in `apps/api/shared`** points to a path on a different developer's machine (`/Users/varshini/...`). It is dead on this machine; harmless because the production API uses `app/shared_roles.py` directly and never imports through that symlink.
- **`uploaded_by` vs `uploaded_by_user_id`.** The migration defines `uploaded_by_user_id`, but the upload handler writes `uploaded_by` and the read handlers read `uploaded_by`. Either the live Supabase schema diverges from the Alembic migration, or this is an existing bug worth checking before touching either path.
- **`is_superadmin`** short-circuits project membership checks ([apps/api/app/deps.py:62-69](apps/api/app/deps.py#L62-L69)) — they get a synthetic admin membership for any project.
- **Citations are persisted** in `chat_messages.citations_json`, so reopening a session re-renders the same source links without re-running RAG.
- **Audit log** records: `auth.register`, `auth.login`, `project.create`, `member.add`, `member.update`, `document.upload`, `document.view`, `chat.query`. Visible to admins and project_managers via `/api/projects/{id}/audit`.

---

## 14. Quick reference — file index

| Concern | File |
|---------|------|
| FastAPI entry | [apps/api/app/main.py](apps/api/app/main.py) |
| Vercel handler | [apps/api/api/index.py](apps/api/api/index.py) |
| Settings | [apps/api/app/config.py](apps/api/app/config.py) |
| Auth dependencies | [apps/api/app/deps.py](apps/api/app/deps.py) |
| JWT | [apps/api/app/security/jwt.py](apps/api/app/security/jwt.py) |
| Roles & RBAC maps | [apps/api/app/shared_roles.py](apps/api/app/shared_roles.py) |
| RBAC scope helper | [apps/api/app/rbac/filters.py](apps/api/app/rbac/filters.py) |
| Ingestion (serverless) | [apps/api/app/services/ingestion.py](apps/api/app/services/ingestion.py) |
| Ingestion (Celery) | [apps/worker/worker/tasks/ingest.py](apps/worker/worker/tasks/ingest.py) |
| Chat orchestration | [apps/api/app/services/chat_service.py](apps/api/app/services/chat_service.py) |
| Hybrid retrieval | [apps/api/app/retrieval/hybrid_search.py](apps/api/app/retrieval/hybrid_search.py) |
| pgvector RPC client | [apps/api/app/retrieval/qdrant_store.py](apps/api/app/retrieval/qdrant_store.py) |
| Generator + system prompt | [apps/api/app/ai/generator.py](apps/api/app/ai/generator.py) |
| LLM providers | [apps/api/app/ai/provider.py](apps/api/app/ai/provider.py) |
| Embedding | [apps/api/app/ai/embeddings.py](apps/api/app/ai/embeddings.py) |
| Reranker | [apps/api/app/ai/reranker.py](apps/api/app/ai/reranker.py) |
| Storage | [apps/api/app/services/storage.py](apps/api/app/services/storage.py) |
| Audit | [apps/api/app/services/audit_service.py](apps/api/app/services/audit_service.py) |
| DB schema | [apps/api/alembic/versions/001_initial_schema.py](apps/api/alembic/versions/001_initial_schema.py) |
| Web API client | [apps/web/src/lib/api-client.ts](apps/web/src/lib/api-client.ts) |
| Web rewrites | [apps/web/next.config.ts](apps/web/next.config.ts) |
| Compose stack | [docker-compose.yml](docker-compose.yml) |
| Render deploy | [render.yaml](render.yaml) |
| Vercel deploy (API) | [apps/api/vercel.json](apps/api/vercel.json) |
| Demo seed | [infra/scripts/seed.py](infra/scripts/seed.py) |
