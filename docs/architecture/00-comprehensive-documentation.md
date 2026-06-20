# StratLogic Open WebUI — Comprehensive Technical Documentation

> **Version:** v0.9.6 | **Branch:** stratlogic-product | **Generated:** June 2026
> **Upstream Fork:** Open WebUI (https://github.com/open-webui/open-webui)
> **Scope:** Full-stack SaaS AI chat platform with multi-tenant architecture

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Project Overview](#2-project-overview)
3. [Technology Stack](#3-technology-stack)
4. [System Architecture](#4-system-architecture)
5. [Frontend Architecture](#5-frontend-architecture)
6. [Backend Architecture](#6-backend-architecture)
7. [API Documentation](#7-api-documentation)
8. [Database Documentation](#8-database-documentation)
9. [Entity Relationship Documentation](#9-entity-relationship-documentation)
10. [Business Domain Model](#10-business-domain-model)
11. [Authentication & Authorization](#11-authentication--authorization)
12. [Multi-Tenant Architecture](#12-multi-tenant-architecture)
13. [Credit & Billing Architecture](#13-credit--billing-architecture)
14. [AI Architecture](#14-ai-architecture)
15. [Infrastructure Architecture](#15-infrastructure-architecture)
16. [Queue & Scheduler Architecture](#16-queue--scheduler-architecture)
17. [Security Documentation](#17-security-documentation)
18. [Performance Overview](#18-performance-overview)
19. [Dependency Mapping](#19-dependency-mapping)
20. [Configuration Guide](#20-configuration-guide)
21. [Deployment Guide](#21-deployment-guide)
22. [Data Flow Documentation](#22-data-flow-documentation)
23. [Component Relationship Documentation](#23-component-relationship-documentation)

---

## 1. Executive Summary

StratLogic Open WebUI is a full-stack, multi-tenant SaaS AI chat platform forked from Open WebUI v0.9.6. It provides a complete platform for organizations to deploy, manage, and use Large Language Models (LLMs) with enterprise-grade features including multi-tenancy, RBAC, credit/billing, RAG pipeline, real-time collaboration, and extensive AI provider integrations.

The system is a **monorepo** containing:
- **Backend:** FastAPI (Python 3.11) — ~3029 lines in main.py, 30 API routers, 37 database tables across 25 model files
- **Frontend:** SvelteKit SPA (TypeScript, Svelte 5) — 46 route pages, 55 writable stores, 30 API client modules, ~530+ components
- **Database:** SQLAlchemy 2.0 with dual-engine (sync/async), SQLite default with optional PostgreSQL
- **AI Pipeline:** 5272-line chat processing pipeline, 30 web search engines, 15 vector DB backends, 4 embedding engines

**Scale:** ~600 source files, ~120K+ lines of code across backend and frontend.

**Key Architectural Decisions:**
1. **Dual-write chat storage:** Chat history stored as both JSON blob (backward compatibility) and normalized `chat_message` table (queryability)
2. **Dynamic code loading:** User-defined functions/tools loaded at runtime via `exec()` rather than a traditional plugin framework
3. **Unified middleware chain:** 10-layer middleware stack combining Starlette built-ins with custom pure-ASGI middlewares
4. **Polymorphic access grants:** Per-resource access control via `access_grants` table supporting user, group, and public wildcard principals
5. **Hybrid real-time:** Socket.IO for live updates with Redis pubsub for cross-instance coordination

---

## 2. Project Overview

### Business Purpose

Open WebUI is an open-source, self-hosted AI chat interface that serves as a unified frontend for multiple LLM providers. StratLogic extends this with:

- **Multi-tenant SaaS:** Organizations with isolated data, users, and AI contexts
- **RBAC system:** Role-based access control with granular permissions
- **Credit/billing:** Per-organization credit system with transaction history
- **Admin panel:** Full administrative control over users, settings, analytics, and evaluations
- **Chatbot embedding:** Embeddable chat widgets for external websites

### Core Modules

```
┌─────────────────────────────────────────────────────────────┐
│                    STRATLOGIC OPEN WEBUI                     │
├─────────────────────────────────────────────────────────────┤
│  Frontend (SvelteKit SPA)                                   │
│  ┌──────────┬──────────┬──────────┬──────────┬──────────┐  │
│  │   Chat   │ Admin    │Workspace │ Calendar │ Channels │  │
│  │ Interface│  Panel   │  (CRUD)  │          │          │  │
│  └──────────┴──────────┴──────────┴──────────┴──────────┘  │
├─────────────────────────────────────────────────────────────┤
│  Backend (FastAPI)                                          │
│  ┌──────────┬──────────┬──────────┬──────────┬──────────┐  │
│  │   Auth   │   Chat   │  Model   │   RAG    │Knowledge │  │
│  │   RBAC   │ Pipeline │  Proxy   │ Pipeline │   Base   │  │
│  └──────────┴──────────┴──────────┴──────────┴──────────┘  │
├─────────────────────────────────────────────────────────────┤
│  Infrastructure                                             │
│  ┌──────────┬──────────┬──────────┬──────────┬──────────┐  │
│  │ SQLite/  │  Redis   │  Vector  │ Storage  │  Socket  │  │
│  │PostgreSQL│ (cache)  │   DBs    │  (S3...) │   .IO    │  │
│  └──────────┴──────────┴──────────┴──────────┴──────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Main Features

- Multi-provider AI chat (Ollama, OpenAI-compatible, Anthropic, Google, custom pipelines)
- RAG (Retrieval-Augmented Generation) with 15 vector DB backends
- Web search integration (30 search engines)
- Real-time collaborative channels with threads
- Knowledge base management with document ingestion
- Code interpreter (Jupyter kernel or Pyodide WASM)
- Tool/Function/Skill system with dynamic code loading
- Calendar with iCalendar RRULE support and event attendees
- Automation scheduler for recurring LLM tasks
- Notes with rich text editor
- Prompt versioning with git-like history
- OAuth/OIDC integration (Google, Microsoft, GitHub, and custom providers)
- SCIM 2.0 provisioning
- OpenTelemetry observability (traces, metrics, logs)
- Audit logging with configurable levels
- 63 i18n locale translations

### Project Structure

```
open-webui/
├── backend/
│   ├── open_webui/
│   │   ├── main.py              # FastAPI app + 436 endpoints (3029 lines)
│   │   ├── config.py            # Persistent config + 250+ ConfigVars (4047 lines)
│   │   ├── env.py               # Environment variable definitions (1069 lines)
│   │   ├── routers/             # 30 API routers
│   │   ├── models/              # 25 model files (37 tables)
│   │   ├── utils/               # Utilities: middleware, auth, RAG, tools, etc.
│   │   ├── retrieval/           # RAG: loaders, vector DBs, web search
│   │   ├── socket/              # Socket.IO event handlers
│   │   ├── storage/             # File storage providers
│   │   ├── tools/               # Builtin tools (3617 lines)
│   │   ├── migrations/          # 44 Alembic migration versions
│   │   └── static/              # Static files (served in production)
│   └── data/                    # Runtime data (SQLite, uploads, vector_db)
├── src/                         # SvelteKit frontend
│   ├── routes/                  # File-based routing (46 +page.svelte)
│   ├── lib/
│   │   ├── apis/               # 30 API client modules
│   │   ├── stores/             # 55 writable Svelte stores
│   │   ├── components/         # ~530 .svelte components
│   │   ├── i18n/               # 63 locale translations
│   │   └── utils/              # Frontend utilities
├── build/                       # Compiled frontend (after npm run build)
├── Dockerfile                   # Multi-stage Docker build
├── docker-compose.yaml          # Main deployment + 7 overlay files
├── pyproject.toml              # Python dependencies (261 packages)
├── package.json                 # Node.js dependencies
└── .env                         # Environment configuration
```

---

## 3. Technology Stack

### Backend

| Category | Technology | Version |
|----------|-----------|---------|
| **Framework** | FastAPI | 0.135.1 |
| **Server** | Uvicorn | 0.41.0 |
| **ORM** | SQLAlchemy (async) | 2.0.48 |
| **Migrations** | Alembic | 1.18.4 |
| **Auth** | python-jose (JWT), PyJWT, bcrypt, argon2 | — |
| **OAuth** | Authlib | 1.6.10 |
| **Database** | SQLite (aiosqlite 0.21.0) / PostgreSQL (psycopg 3.2.9) | — |
| **Cache** | Redis | 7.4.0 |
| **WebSocket** | python-socketio (ASGI) | 5.16.1 |
| **HTTP Client** | aiohttp, httpx, requests | — |
| **AI SDKs** | openai, anthropic, google-genai | — |
| **RAG/ML** | sentence-transformers, chromadb, langchain | — |
| **Document Processing** | pypdf, python-pptx, docx2txt, openpyxl | — |
| **Cloud Storage** | boto3, google-cloud-storage, azure-storage-blob | — |
| **Observability** | OpenTelemetry + loguru | — |
| **Task Scheduling** | APScheduler, custom scheduler worker | — |

### Frontend

| Category | Technology | Version |
|----------|-----------|---------|
| **Framework** | SvelteKit (SPA mode) | 2.5.27 |
| **UI** | Svelte 5 (runes mode) | 5.53.10 |
| **Styling** | Tailwind CSS v4 | 4.0.0 |
| **State** | Svelte writable stores (55 stores) | — |
| **Build** | Vite | 5.4.21 |
| **Language** | TypeScript | 5.5.4 |
| **Real-time** | Socket.IO Client | 4.2.0 |
| **Markdown** | marked + highlight.js + shiki + katex + mermaid | — |
| **Rich Text** | Tiptap (ProseMirror) | 3.0.7 |
| **UI Primitives** | bits-ui, floating-ui, tippy.js, sortablejs | — |
| **i18n** | i18next | 23.10.0 |
| **Charts** | chart.js, @xyflow/svelte, vega-lite | — |

### Infrastructure

| Category | Technology |
|----------|-----------|
| **Containerization** | Docker (multi-stage, python:3.11-slim-bookworm + node:22-alpine) |
| **Orchestration** | Docker Compose (8 compose files for different use cases) |
| **Reverse Proxy** | Nginx (recommended, not included) |
| **LLM Serving** | Ollama, OpenAI-compatible APIs, custom pipelines |
| **Vector Storage** | Chroma (default), Qdrant, Milvus, Pinecone, Elasticsearch, etc. |
| **File Storage** | Local filesystem, AWS S3, GCS, Azure Blob |
| **Monitoring** | OpenTelemetry → Grafana LGTM stack |

---

## 4. System Architecture

### High-Level Architecture Diagram

```
                      ┌──────────────────────┐
                      │     Browser / App     │
                      │  (SvelteKit SPA)      │
                      │  SPA: ssr=false       │
                      └──────┬───────┬───────┘
                             │ HTTP  │ WebSocket
                             ▼       ▼
┌─────────────────────────────────────────────────────────┐
│                   MIDDLEWARE CHAIN (10 layers)           │
│  Compress → Redirect → SecurityHeaders → CommitSession  │
│  → AuthToken → WebsocketGuard → CORS → Session → Audit │
├─────────────────────────────────────────────────────────┤
│                      FASTAPI APP                         │
│  ┌─────────────┬─────────────┬──────────────────────┐  │
│  │ 30 Routers  │ Auth System │ Chat Pipeline (5272L) │  │
│  │ 436 APIs    │ JWT/OAuth   │ RAG + Tools + Search │  │
│  └─────────────┴─────────────┴──────────────────────┘  │
├─────────────────────────────────────────────────────────┤
│                    DATA LAYER                            │
│  ┌──────────┬──────────┬──────────┬──────────────────┐ │
│  │ SQLite/  │  Redis   │  Vector  │  Storage (S3/    │ │
│  │PostgreSQL│  Cache   │  DB (15) │  GCS/Azure/Local)│ │
│  └──────────┴──────────┴──────────┴──────────────────┘ │
├─────────────────────────────────────────────────────────┤
│                  BACKGROUND JOBS                         │
│  ┌──────────┬──────────┬──────────┬──────────────────┐ │
│  │Scheduler │ Task Mgr │ Cleanup  │ Prefetch         │ │
│  │Worker    │ (pubsub) │ Workers  │ Services         │ │
│  └──────────┴──────────┴──────────┴──────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### Request Lifecycle

1. **HTTP Request** arrives at Uvicorn ASGI server
2. **CompressMiddleware** — optionally compresses response
3. **RedirectMiddleware** — rewrites legacy URLs to SPA routes
4. **SecurityHeadersMiddleware** — injects CSP, HSTS, X-Frame-Options headers
5. **CommitSessionMiddleware** — manages sync DB session lifecycle per request
6. **AuthTokenMiddleware** — extracts bearer token, cookie, or x-api-key to `request.state.token`
7. **WebsocketUpgradeGuardMiddleware** — rejects non-WebSocket requests to WS endpoints
8. **CORSMiddleware** — handles cross-origin requests
9. **SessionMiddleware** (Redis or cookie) — loads OAuth session state
10. **AuditLoggingMiddleware** — conditionally logs request/response metadata
11. **FastAPI Router** — matches URL pattern, resolves dependencies
12. **Auth Dependency** (`get_verified_user` / `get_admin_user`) — validates JWT/API key
13. **Business Logic** — router handler executes, queries models, processes data
14. **Response** — returned through middleware chain (commit/release/compress)

---

## 5. Frontend Architecture

### Framework: SvelteKit SPA

The frontend is a **single-page application** built with SvelteKit configured as a static site with SPA fallback (`ssr=false`).

```
Configuration:
- Adapter: @sveltejs/adapter-static (output: build/)
- SSR: disabled (runs entirely client-side)
- Svelte 5 with $props() runes syntax
- Tailwind CSS v4 (class-based dark mode)
- TypeScript with strict mode
```

### Route Architecture

**46 route pages, 7 layouts** organized as:

```
routes/
├── +layout.svelte                    # Root: i18n, Socket.IO, theme, auth check
├── auth/+page.svelte                 # Public: sign-in/sign-up/OAuth/LDAP
├── s/[id]/+page.svelte              # Public: shared chat viewer
├── watch/+page.svelte                # Public: shared chat watch
├── error/+page.svelte                # Public: backend error page
│
├── (app)/+layout.svelte             # Auth guard + Sidebar + data loading
│   ├── +page.svelte                  # Main chat interface
│   ├── c/[id]/+page.svelte          # Specific chat
│   │
│   ├── admin/+layout.svelte          # Admin-only guard + top nav
│   │   ├── +page.svelte
│   │   ├── users/+page.svelte
│   │   ├── users/[tab]/+page.svelte
│   │   ├── analytics/+page.svelte
│   │   ├── analytics/[tab]/+page.svelte
│   │   ├── evaluations/+page.svelte
│   │   ├── evaluations/[tab]/+page.svelte
│   │   ├── functions/+page.svelte
│   │   ├── functions/create/+page.svelte
│   │   ├── functions/edit/+page.svelte
│   │   ├── settings/+page.svelte
│   │   └── settings/[tab]/+page.svelte
│   │
│   ├── workspace/+layout.svelte       # Workspace shell
│   │   ├── knowledge/...             # Knowledge base CRUD
│   │   ├── models/...                # Model configuration
│   │   ├── prompts/...               # Prompt management
│   │   ├── skills/...                # Skill CRUD
│   │   ├── tools/...                 # Tool management
│   │   └── functions/create/...      # Function creation
│   │
│   ├── channels/[id]/+page.svelte    # Real-time channels
│   ├── notes/...                     # Notes feature
│   ├── playground/...                # AI playground
│   ├── calendar/+page.svelte         # Calendar
│   ├── automations/...               # Automation editor
│   └── home/...                      # Home dashboard
```

### State Management: 55 Writable Stores

All state is managed through Svelte `writable()` stores in `src/lib/stores/index.ts`. No derived stores — components use `$:` reactive declarations for computed values.

**Core Stores (5):**
| Store | Type | Purpose |
|-------|------|---------|
| `user` | SessionUser | Current authenticated user |
| `config` | Config | Backend feature flags, OAuth providers |
| `WEBUI_NAME` | string | Application display name |
| `WEBUI_VERSION` | string | Backend-reported version |
| `WEBUI_DEPLOYMENT_ID` | string | Deployment identifier |

**Chat Stores (13):**
| Store | Type | Purpose |
|-------|------|---------|
| `chats` | array | Full chat list |
| `chatId` | string | Currently active chat ID |
| `chatTitle` | string | Current chat title |
| `pinnedChats` | array | Pinned chat references |
| `pinnedNotes` | array | Pinned note references |
| `tags` | array | All chat tags |
| `folders` | array | Chat folder tree |
| `selectedFolder` | any | Currently selected folder |
| `activeChatIds` | Set | Chat IDs with active connections |
| `temporaryChatEnabled` | boolean | Temp chat mode |
| `chatRequestQueues` | Record | Queued messages per chat |
| `scrollPaginationEnabled` | boolean | Infinite scroll pagination |
| `currentChatPage` | number | Current pagination page |

**Resource Stores (10):**
| Store | Type | Purpose |
|-------|------|---------|
| `models` | Model[] | Available AI models |
| `knowledge` | Document[] | Knowledge base documents |
| `tools` | array | Available tools |
| `skills` | array | Available skills |
| `functions` | array | Available functions |
| `toolServers` | array | Connected tool servers |
| `terminalServers` | array | Connected terminal servers |
| `channels` | array | Real-time channels |
| `channelId` | string | Active channel ID |
| `banners` | Banner[] | UI banner announcements |

**UI Stores (27):**
Window states (`showSidebar`, `showSettings`, `showSearch`, `showShortcuts`, `showArchivedChats`, `showChangelog`, `showControls`, `showEmbeds`, `showOverview`, `showArtifacts`, `showCallOverlay`, `showFileNav`), theme, mobile detection, sidebar width, file nav state, artifact content, embed data, etc.

**Realtime/Electron Stores (10):**
Socket.IO instance, connection state, active users, usage pool, model download pool, Electron app state, TTS worker, Pyodide worker, audio queue, etc.

### API Client Architecture

**30 modules** in `src/lib/apis/`:

Pattern:
```typescript
// Each module follows this exact pattern
async function apiCall(token: string, ...params) {
    let error = null;
    const res = await fetch(`${WEBUI_API_BASE_URL}/endpoint`, {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            ...(token && { authorization: `Bearer ${token}` })
        },
        body: JSON.stringify({...})
    }).then(async (res) => {
        if (!res.ok) throw await res.json();
        return res.json();
    }).catch((err) => {
        error = err.detail;
        return null;
    });
    if (error) throw error;
    return res;
}
```

**Auth token:** Passed as function parameter (typically `localStorage.token`). In production build, `WEBUI_API_BASE_URL` is empty string (same-origin).

**Streaming module** (`apis/streaming/index.ts`): Uses `eventsource-parser` for SSE streaming with delta chunking, source/citation extraction, and usage tracking.

### Component Categories (~530+ .svelte files)

| Category | Count | Purpose |
|----------|-------|---------|
| `chat/` | 126 | Chat UI: Messages, Input, Markdown, ModelSelector, Settings, FileNav, Artifacts |
| `icons/` | 178 | SVG icon components (Heroicons-style) |
| `admin/` | 58 | Admin panel: Users, Settings, Functions, Analytics, Evaluations |
| `common/` | 49 | Reusable: Modal, Dropdown, Spinner, CodeEditor, PDFViewer, ConfirmDialog |
| `workspace/` | 45 | CRUD editors: Models, Knowledge, Prompts, Skills, Tools |
| `layout/` | 22 | Layout chrome: Sidebar (1659 lines), Navbar, SearchModal, ChatItem |
| `channel/` | 17 | Real-time channels: Messages, Thread, Webhooks, PinnedMessages |
| `notes/` | 10 | Notes: Editor, Panel, Chat integration |
| `calendar/` | 5 | Calendar: View, Sidebar, EventModal |
| `automations/` | 5 | Automation: Editor, Menu, Dropdowns |
| `playground/` | 5 | Playground: Chat, Completions, Images |

### Layout System

**Root `+layout.svelte` (1213 lines):**
- Initializes i18next, Socket.IO, theme, Electron detection
- Fetches backend config, validates session, sets up Socket.IO event handlers
- Auth routing: no token → `/auth`, bad session → `/auth`, no backend → `/error`
- Multi-tab coordination via `BroadcastChannel`
- Sets up `checkTokenExpiry` timer (15s)

**`(app)/+layout.svelte` (517 lines):**
- Auth guard: `$user` null → goto `/auth`; pending role → `AccountPending` overlay
- Loads: settings, models, tool servers, tools, banners
- Keyboards shortcuts (15+): SEARCH, NEW_CHAT, FOCUS_INPUT, TOGGLE_SIDEBAR, etc.
- Renders: Sidebar + `<slot/>` (with Spinner until loaded)

**Sidebar (`layout/Sidebar.svelte` — 1659 lines):**
- Drag-to-resize (persisted to localStorage)
- New chat button, folder tree (SortableJS), chat list (paginated), pinned models/chats
- Channel list, user menu, temporary chat toggle
- Chat CRUD, folder CRUD, search, import/export

### i18n (Internationalization)

- **Library:** i18next v23.10.0
- **63 supported locales** including Indonesian (id-ID)
- Detection order: `querystring → localStorage → navigator`
- Namespace: single `translation` per locale
- Auto-extraction via `i18next-parser` from `src/**/*.{js,svelte}`

### Auth Flow (Frontend)

```
1. Sign-in → POST /api/v1/auths/signin → {token, user}
2. localStorage.token = token
3. Socket.IO emit('user-join', {auth: {token}})
4. user.set(sessionUser), config.set(backendConfig)
5. goto(redirectPath || '/')
6. Root layout: getSessionUser(token) → validates token is still valid
7. (app) layout: checks $user is authenticated + non-pending
8. Socket.IO auth: io({auth: {token}}, path: '/ws/socket.io')
9. Socket events: chat completions, channel messages, user presence
10. Token expiry: checked every 15s, redirects to /auth if expired
```

---

## 6. Backend Architecture

### App Factory

FastAPI application created at module level in `main.py:749`:

```python
app = FastAPI(
    title='Open WebUI',
    docs_url='/docs' if ENV == 'dev' else None,
    openapi_url='/openapi.json' if ENV == 'dev' else None,
    redoc_url=None,
    lifespan=lifespan,
)
```

**Lifespan startup sequence (17 steps):**
1. Store event loop reference
2. Set instance ID
3. Start structured logger
4. Optional: reset config on start
5. Validate license key
6. Auto-create admin user (from env vars)
7. Optional: deactivate all functions in safe mode
8. Install tool/function dependencies (pip install from frontmatter)
9. Connect Redis → start task command listener (pubsub)
10. Set thread pool size limit
11. Start periodic usage pool cleanup (every 3s)
12. Start periodic session pool cleanup (every 120s)
13. Start scheduler worker loop (automations + calendar alerts)
14. Optional: pre-fetch models for cache
15. Pre-initialize tool + terminal servers
16. Set `startup_complete = True`

**Health endpoints:**
- `/health` — always returns `{"status": True}`
- `/ready` — checks `startup_complete`, DB ping, Redis ping → 200 or 503
- `/health/db` — DB ping only

### Middleware Chain (10 layers)

Registered in order at `main.py:1382-1473`:

| # | Middleware | Type | Conditional | Function |
|---|-----------|------|-------------|----------|
| 1 | CompressMiddleware | Starlette | `ENABLE_COMPRESSION_MIDDLEWARE` | GZip/Brotli compression |
| 2 | RedirectMiddleware | Pure ASGI | Always | Legacy URL rewriting |
| 3 | SecurityHeadersMiddleware | BaseHTTP | Always | CSP, HSTS, X-Frame-Options |
| 4 | CommitSessionMiddleware | Pure ASGI | Always | Sync DB session lifecycle |
| 5 | AuthTokenMiddleware | Pure ASGI | Always | Token extraction + timing |
| 6 | WebsocketUpgradeGuardMiddleware | Pure ASGI | Always | WS endpoint validation |
| 7 | CORSMiddleware | Starlette | Always | Cross-origin support |
| 8 | SessionAutoloadMiddleware | StarSessions | `ENABLE_STAR_SESSIONS_MIDDLEWARE` | Redis session loading |
| 9 | StarSessionsMiddleware | StarSessions | `ENABLE_STAR_SESSIONS_MIDDLEWARE` | Redis session storage |
| 10 | AuditLoggingMiddleware | Pure ASGI | `AUDIT_LOG_LEVEL != NONE` | Audit logging |

### Router Inventory (30 routers, 436 total endpoints)

**Auth Group:**
| Router | Prefix | Lines | Condition |
|--------|--------|-------|-----------|
| auths.py | `/api/v1/auths` | 1393 | — |
| scim.py | `/api/v1/scim/v2` | 1012 | `ENABLE_SCIM` |

**Chat Group:**
| Router | Prefix | Lines |
|--------|--------|-------|
| chats.py | `/api/v1/chats` | 1578 |
| channels.py | `/api/v1/channels` | 1838 |
| notes.py | `/api/v1/notes` | 500 |

**Resources Group:**
| Router | Prefix | Lines |
|--------|--------|-------|
| models.py | `/api/v1/models` | 804 |
| knowledge.py | `/api/v1/knowledge` | 1584 |
| prompts.py | `/api/v1/prompts` | 710 |
| tools.py | `/api/v1/tools` | 931 |
| skills.py | `/api/v1/skills` | 453 |
| functions.py | `/api/v1/functions` | 563 |
| files.py | `/api/v1/files` | 907 |
| folders.py | `/api/v1/folders` | 321 |
| memories.py | `/api/v1/memories` | 360 |
| images.py | `/api/v1/images` | 1125 |
| audio.py | `/api/v1/audio` | 1400 |
| pipelines.py | `/api/v1/pipelines` | 533 |
| retrieval.py | `/api/v1/retrieval` | 2730 |
| evaluations.py | `/api/v1/evaluations` | 426 |
| calendar.py | `/api/v1/calendars` | 419 |
| automations.py | `/api/v1/automations` | 303 |
| terminals.py | `/api/v1/terminals` | 355 |
| tasks.py | `/api/v1/tasks` | 712 |

**Admin Group:**
| Router | Prefix | Lines | Condition |
|--------|--------|-------|-----------|
| users.py | `/api/v1/users` | 761 | — |
| groups.py | `/api/v1/groups` | 351 | — |
| configs.py | `/api/v1/configs` | 674 | — |
| analytics.py | `/api/v1/analytics` | 442 | `ENABLE_ADMIN_ANALYTICS` |

**Infrastructure Group:**
| Router | Prefix | Lines |
|--------|--------|-------|
| ollama.py | `/ollama` | 1530 |
| openai.py | `/openai` | 1619 |
| utils.py | `/api/v1/utils` | 111 |

### Auth System

**JWT (JSON Web Token):**
- Algorithm: HS256 (HMAC-SHA256)
- Secret: `WEBUI_SECRET_KEY` (required env var)
- Payload: `id`, `exp`, `jti` (UUID4 token ID), `iat`
- Expiry: configurable via `JWT_EXPIRES_IN`
- Revocation: per-token (Redis `auth:token:{jti}:revoked`) or per-user (OIDC back-channel `auth:user:{id}:revoked_at`)

**Auth Dependencies:**
- `get_current_user` — raw token extraction + DB lookup (bearer, cookie, x-api-key)
- `get_verified_user` — `get_current_user` + role check (user/admin only, rejects pending)
- `get_admin_user` — `get_current_user` + `role == 'admin'`

**API Key Auth:**
- Keys generated as `sk-{uuid4().hex}`
- Detected by `sk-` prefix in `get_current_user`
- Optional endpoint restrictions via `API_KEYS_ALLOWED_ENDPOINTS`
- Transport: `Authorization: Bearer sk-...`, `token` cookie, or `x-api-key` header

**OAuth Flow:**
- `/oauth/{provider}/login` → redirect to provider
- `/oauth/{provider}/login/callback` → exchange code, resolve/create user
- User resolution: `sub` claim match → email match (`OAUTH_MERGE_ACCOUNTS_BY_EMAIL`) → create new (`ENABLE_OAUTH_SIGNUP`)
- OAuth role management: apply role from `OAUTH_ROLES_CLAIM`
- OAuth group management: apply groups from `OAUTH_GROUPS_CLAIM`

**Session Management:**
- Preferred: Redis-backed via StarSessions (`owui-session` cookie)
- Fallback: Starlette signed cookie session (in-memory)

### RBAC System

**Three roles:** `admin`, `user`, `pending`

**Permissons defined in `config.py` `DEFAULT_USER_PERMISSIONS`:**
- `workspace`: models, knowledge, prompts, tools, skills + import/export
- `sharing`: per-resource-type public/private sharing
- `access_grants`: allow_users
- `chat`: controls, valves, system_prompt, params, file_upload, web_upload, delete, edit, share, export, stt, tts, call, multiple_models, temporary
- `features`: api_keys, notes, folders, channels, web_search, image_generation, code_interpreter, memories, automations, calendar
- `settings`: interface

**Permission resolution:** Base defaults → merge group permissions (most-permissive wins) → fill missing keys. Admins bypass all permission checks.

**Access Grants (resource-level):**
Polymorphic table `access_grants(resource_type, resource_id, principal_type, principal_id, permission)`. Used by 9 resource types: knowledge, model, prompt, tool, note, channel, file, calendar, skill.

### Plugin System (Dynamic Code Loading)

Not a traditional plugin framework — user-written Python code loaded at runtime via `exec()`:

- **Functions:** `Pipe` (transform requests), `Filter` (pipeline hooks), `Action` (side effects)
- **Tools:** Python classes with `__call__` exposed as OpenAI tools
- **Valves:** Configurable parameters via Pydantic models (`Valves` and `UserValves`)
- **Manifold:** A `Pipe` with `pipes` attribute exposes sub-pipes as individual model IDs
- **Caching:** `app.state.FUNCTIONS`, `app.state.TOOLS` caches with content hash-based invalidation
- **Dependency installation:** Frontmatter `requirements` auto-installed at startup

---

## 7. API Documentation

### API Groups Overview

The platform exposes **436 REST endpoints** across 30 routers:

**Auth APIs** (`/api/v1/auths` — 21 endpoints):
- `POST /signup` — Register new user
- `POST /signin` — Login with email/password
- `POST /signout` — Revoke token
- `GET /session` — Get current session user
- `POST /token/refresh` — Refresh JWT
- `POST /update/password` — Change password
- `POST /update/profile` — Update profile
- `POST /api_key` — Generate API key
- `DELETE /api_key/{id}` — Delete API key
- `GET /api_key` — List API keys

**Chat APIs** (`/api/v1/chats` — 41 endpoints):
- `GET /` — List user's chats (paginated)
- `POST /new` — Create new chat
- `GET /{id}` — Get chat by ID
- `POST /{id}` — Update chat
- `DELETE /{id}` — Delete chat
- `POST /{id}/archive` — Archive chat
- `POST /{id}/share` — Share chat
- `GET /{id}/messages` — Get chat messages
- `POST /{id}/pin` / `DELETE /{id}/pin` — Pin/unpin chat
- `POST /{id}/clone` — Clone chat
- `POST /import` — Import chat
- `GET /export` — Export all chats
- `GET /export/{id}` — Export specific chat
- `GET /tags/list` — List all tags
- `POST /tags` — Create/delete tags
- `GET /folders/list` — List folders
- `POST /folders` — Create folder
- `PUT /folders/{id}` — Update folder
- `DELETE /folders/{id}` — Delete folder
- `GET /pinned` — Get pinned chats

**Model APIs** (`/api/v1/models` — 14 endpoints):
- `GET /` — List models
- `POST /create` — Create model configuration
- `GET /{id}` — Get model
- `POST /{id}/update` — Update model
- `DELETE /{id}` — Delete model
- `POST /{id}/share` — Share model
- `POST /{id}/export` — Export model
- `POST /import` — Import model

**Knowledge APIs** (`/api/v1/knowledge` — 24 endpoints):
- `GET /` — List knowledge bases
- `POST /create` — Create knowledge base
- `GET /{id}` — Get knowledge base
- `POST /{id}/update` — Update knowledge base
- `DELETE /{id}` — Delete knowledge base
- `POST /{id}/file/add` — Add file to knowledge base
- `POST /{id}/file/remove` — Remove file
- `GET /{id}/files` — List files
- `POST /{id}/text/add` — Add text content
- `POST /{id}/query` — Query knowledge base
- `GET /{id}/directory` — List directory structure
- `POST /{id}/directory/create` — Create directory
- `PUT /{id}/directory/{dir_id}` — Update directory

**Retrieval APIs** (`/api/v1/retrieval` — 16 endpoints):
- `POST /process/file` — Process file for RAG
- `POST /process/web` — Process web URL
- `POST /process/text` — Process text
- `POST /query/collection` — Query vector collection
- `POST /query/doc` — Query specific document
- `POST /search` — Semantic search
- `POST /web/search` — Web search (30 engines)
- `POST /ef` — Get embedding function
- `POST /template` — Get RAG template
- `GET /config` — Get retrieval config
- `POST /config/update` — Update retrieval config
- `POST /{id}/delete` — Delete from vector DB
- `POST /{id}/reset` — Reset collection
- `POST /{id}/upload` — Upload for processing via loader

**Resource CRUD APIs (similar pattern for each):**
- `prompts/` (15 endpoints), `tools/` (15), `skills/` (9), `functions/` (17)
- `files/` (13), `folders/` (7), `memories/` (7), `images/` (6), `audio/` (6)
- `notes/` (9), `evaluations/` (15), `pipelines/` (8), `utils/` (5)

**Admin APIs:**
- `users/` (22 endpoints) — User CRUD, import, bulk operations
- `groups/` (11 endpoints) — Group CRUD, member management, permissions
- `configs/` (20 endpoints) — System configuration (read/write)
- `analytics/` (8 endpoints) — Usage stats: users, models, messages, tokens

**Infrastructure APIs:**
- `/ollama` (43 endpoints) — Ollama API proxy + management
- `/openai` (8 endpoints) — OpenAI-compatible API proxy
- `/scim/v2` (15 endpoints, conditional) — SCIM 2.0 provisioning

**Global APIs (in main.py):**
- `POST /api/chat/completions` — Main chat endpoint (streaming SSE)
- `POST /api/embeddings` — Embedding endpoint
- `GET /api/models` — Model listing
- `GET /oauth/{provider}/login` — OAuth login
- `GET /oauth/{provider}/login/callback` — OAuth callback
- `GET /manifest.json` — PWA manifest
- `GET /cache/{path}` — Static file cache

**WebSocket:** `/ws/socket.io` — Socket.IO server for real-time events

---

## 8. Database Documentation

### Database Engine

**Dual-engine architecture:**

1. **Sync engine** — startup config, Alembic migrations, health checks
2. **Async engine** — all runtime operations

Default: SQLite with WAL journal mode, 512 connection pool, 64MB cache, 256MB mmap.
Optional: PostgreSQL via `DATABASE_URL` with QueuePool.

**SQLite PRAGMAs:**
- `journal_mode=WAL` (concurrent reads)
- `busy_timeout=5000ms`
- `cache_size=-65536` (64MB)
- `mmap_size=268435456` (256MB)
- `synchronous=NORMAL`
- `temp_store=MEMORY`

**Session settings:** `autocommit=False`, `autoflush=False`, `expire_on_commit=False`

### Table Inventory (37 Tables, 25 Model Files)

| # | File | Table | Columns | PK | Owner |
|---|------|-------|---------|-----|-------|
| 1 | access_grants.py | access_grant | 7 | UUID | — |
| 2 | auths.py | auth | 4 | String | user_id |
| 3 | automations.py | automation | 10 | UUID | user_id |
| 3b | automations.py | automation_run | 6 | UUID | automation_id |
| 4 | calendar.py | calendar | 10 | UUID | user_id |
| 4b | calendar.py | calendar_event | 14 | UUID | user_id |
| 4c | calendar.py | calendar_event_attendee | 8 | UUID | user_id |
| 5 | channels.py | channel | 15 | UUID | user_id |
| 5b | channels.py | channel_member | 15 | UUID | user_id |
| 5c | channels.py | channel_file | 6 | UUID | user_id |
| 5d | channels.py | channel_webhook | 9 | UUID | user_id |
| 6 | chat_messages.py | chat_message | 17 | composite | user_id |
| 7 | chats.py | chat | 14 | String | user_id |
| 7b | chats.py | chat_file | 7 | UUID | user_id |
| 8 | feedbacks.py | feedback | 9 | UUID | user_id |
| 9 | files.py | file | 9 | String | user_id |
| 10 | folders.py | folder | 10 | UUID | user_id |
| 11 | functions.py | function | 11 | String | user_id |
| 12 | groups.py | group | 8 | UUID | user_id |
| 12b | groups.py | group_member | 5 | UUID | user_id |
| 13 | knowledge.py | knowledge | 7 | UUID | user_id |
| 13b | knowledge.py | knowledge_directory | 7 | UUID | user_id |
| 13c | knowledge.py | knowledge_file | 7 | UUID | user_id |
| 14 | memories.py | memory | 5 | String | user_id |
| 15 | messages.py | message | 14 | UUID | user_id |
| 15b | messages.py | message_reaction | 5 | UUID | user_id |
| 16 | models.py | model | 9 | Text | user_id |
| 17 | notes.py | note | 7 | UUID | user_id |
| 17b | notes.py | pinned_note | 4 | UUID | user_id |
| 18 | oauth_sessions.py | oauth_session | 7 | UUID | user_id |
| 19 | prompt_history.py | prompt_history | 7 | UUID | user_id |
| 20 | prompts.py | prompt | 12 | UUID | user_id |
| 21 | shared_chats.py | shared_chat | 7 | UUID | user_id |
| 22 | skills.py | skill | 9 | String | user_id |
| 23 | tags.py | tag | 4 | composite | user_id |
| 24 | tools.py | tool | 9 | String | user_id |
| 25 | users.py | user | 22 | String | — |
| 25b | users.py | api_key | 8 | UUID | user_id |

### Migrations: 44 Alembic Versions

Chain from root to head:
```
7e5b5dc7342b (init) → ca81bd47c050 → c0fbf31ca0db → 6a39f3d8e55c
→ 242a2047eae0 → 1af9b942657b → 3ab32c4b8f59 → c69f45358db4
→ c29facfe716b → af906e964978 → 4ace53fd72c8 → 922e7a387820
→ 57c599a3cb57 → 7826ab40b532 → 3781e22d8b01 → 9f0c9cd09105
→ d31026856c01 → 018012973d35 → 3af16a1c9fb6 → 38d63c18f30f
→ a5c220713937 → 37f288994c47 → 2f1211949ecc → b10670c03dd5
→ 90ef40d4714e → 3e0e00844bb0 → 6283dc0e4d8d → 81cc2ce44d79
→ c440947495f3 → 374d2f66af06 → 8452d01d26d7 → f1e2d3c4b5a6
→ a1b2c3d4e5f6 → b2c3d4e5f6a7 → a3dd5bedd151 → d4e5f6a7b8c9
→ b7c8d9e0f1a2 → e1f2a3b4c5d6 → c1d2e3f4a5b6 → 56359461a091
→ 4de81c2a3af1 → a0b1c2d3e4f5 → 3c9b0ca343fd → 461111b60977 (head)
```

Auto-run on startup if `ENABLE_DB_MIGRATIONS=true`. All migrations are idempotent (check for existing tables/columns before creating).

---

## 9. Entity Relationship Documentation

### Core Entity Map

```
user ────── 1:1 ── auth
user ────── 1:N ── api_key
user ────── 1:N ── oauth_session
user ────── 1:N ── chat
user ────── 1:N ── channel
user ────── 1:N ── group (owner)
user ────── 1:N ── memory, file, folder, function, model, tool, skill, knowledge, prompt, note, feedback, automation, calendar, tag, prompt_history, shared_chat

chat ────── 1:N ── chat_message (CASCADE)
chat ────── 1:N ── chat_file (CASCADE)
chat ────── 1:N ── shared_chat (CASCADE)

channel ─── 1:N ── channel_member
channel ─── 1:N ── channel_file (CASCADE)
channel ─── 1:N ── channel_webhook
channel ─── 1:N ── message

message ─── 1:N ── message_reaction

file ────── 1:N ── chat_file (CASCADE)
file ────── 1:N ── channel_file (CASCADE)
file ────── 1:N ── knowledge_file (CASCADE)

knowledge ── 1:N ── knowledge_directory (CASCADE)
knowledge ── 1:N ── knowledge_file (CASCADE)
knowledge_directory ── 1:N ── knowledge_directory (CASCADE, self-ref)
knowledge_directory ── 1:N ── knowledge_file (SET NULL)

automation ─ 1:N ── automation_run
calendar ─── 1:N ── calendar_event
calendar_event ─ 1:N ── calendar_event_attendee

note ────── 1:N ── pinned_note (CASCADE)
prompt ──── 1:N ── prompt_history (self-ref parent_id)
folder ──── 1:N ── folder (self-ref parent_id)
group ───── 1:N ── group_member ── user

access_grant ── polymorphic to: knowledge, model, prompt, tool, note, channel, file, calendar, skill
               (via resource_type + resource_id)
```

### Key Relationship Notes

- **`auth.id = user.id`**: 1:1 relationship, same primary key
- **`chat_message.id = "{chat_id}-{message_id}"`**: Composite key linking to parent chat
- **`access_grants`**: Polymorphic — `resource_type` + `resource_id` references any of 9 resource tables
- **Self-referential**: `folder.parent_id`, `knowledge_directory.parent_id`, `message.parent_id`, `chat_message.parent_id`, `prompt_history.parent_id`
- **Soft-delete**: `channel.archived_at/deleted_at` + `channel_member.left_at` for channel lifecycle

### Chat Model Dual-Write Architecture

Chat history stored in two formats simultaneously:

1. **JSON blob** (`chat.chat`): Full chat object with `{title, history: {messages: {}, currentId: ...}}`
2. **Normalized table** (`chat_message`): Per-message rows with `{id, chat_id, user_id, role, content, output, model_id, files, sources, usage, ...}`

On read: normalized table preferred (queryable). Falls back to JSON blob for legacy chats.
On write: both formats updated. Background backfill migrates JSON → normalized.

---

## 10. Business Domain Model

### Entity Ownership and Lifecycle

**Users:**
- Created via signup, OAuth, SCIM, or admin creation
- Identified by UUID, authenticated via email/password or OAuth
- Can belong to multiple organizations (multi-tenant) — each membership has a role
- Lifecycle: pending → user/admin → (can be deactivated via `auth.active=false`)

**Organizations (Multi-Tenant):**
- Top-level tenant boundary
- Has name, plan tier, credit balance, branding (logo, colors)
- Contains users via `organization_member` (with roles: owner, admin, member)
- All resources scoped by `organization_id`

**Workspaces:**
- Sub-tenant within organization
- Has name, optional description
- Models, knowledge bases, prompts, tools, skills are workspace-scoped
- Users switch workspaces within their organization

**Groups:**
- Collections of users for access control
- Can have permissions assigned (`permissions` JSON field)
- Used as principals in `access_grants` (principal_type='group')
- Group CRUD managed by admins

**Members:**
- Users within an organization
- Invitation flow: invite → token → accept
- Have roles: owner, admin, member, viewer
- Roles determine permission level (check RBAC system)

**Chats:**
- Single conversation session with an AI model
- Owned by user, scoped to organization
- Contains messages, files, tasks
- Can be shared (generates share token), archived, pinned, tagged, foldered
- Support temporary mode (no persistence)

**Knowledge Bases:**
- Collections of documents for RAG
- Directory tree structure within each knowledge base
- Files attached → processed → chunked → embedded → stored in vector DB
- Supports access grants for sharing

**Models:**
- AI model configurations (provider, parameters, capabilities)
- Can be user-created or system-provided
- Support access grants, valve configuration
- Model capabilities: web_search, image_generation, code_interpreter, etc.

**Credits:**
- Per-organization balance
- Deducted on model usage (based on model pricing)
- Transaction history (`credit_transaction` table)
- Top-up via admin, deduction via usage

**Billing:**
- Tracks credit transactions per organization
- Usage aggregation: per-user, per-model, per-time-period
- Plan tiers determine pricing and feature access

---

## 11. Authentication & Authorization

### Authentication Flow

```
Login Request
    │
    ▼
[POST /api/v1/auths/signin]
    │ email + password
    ▼
Verify credentials (bcrypt/argon2 hash in auth table)
    │
    ▼
Create JWT: {id, exp, jti, iat} signed with WEBUI_SECRET_KEY (HS256)
    │
    ▼
Return {token, user} to client
    │
    ▼
Client stores token in localStorage
    │
    ▼
All subsequent requests: Authorization: Bearer {token}
    │
    ▼
AuthTokenMiddleware extracts token → request.state.token
    │
    ▼
get_verified_user():
  1. Decode JWT → extract user_id
  2. Check revocation (Redis token revocation OR OIDC back-channel)
  3. Load user from DB
  4. Verify role ∈ {user, admin}
  5. Return user object
```

### JWT Lifecycle

- **Creation**: On sign-in, OAuth callback, API key generation
- **Validation**: Every authenticated request via `get_current_user`
- **Revocation**: Explicit sign-out (per-token), OIDC back-channel logout (per-user, time-based)
- **Expiry**: Configurable via `JWT_EXPIRES_IN` (env var)

### RBAC (Role-Based Access Control)

**Roles:**
| Role | Access Level |
|------|-------------|
| `admin` | Full system access, bypasses permission checks |
| `user` | Standard user, governed by permission system |
| `pending` | Blocked — cannot access authenticated endpoints |

**Permission Model:**
- Permissions defined in `DEFAULT_USER_PERMISSIONS` (nested dict)
- Categories: workspace, sharing, access_grants, chat, features, settings
- Assigned to **groups** (not directly to users)
- Users inherit permissions from all their group memberships
- Resolution: base defaults → merge group permissions (OR logic, True wins) → fill gaps

**Access Grants (Resource-Level):**
- Separate from permissions — controls read/write on specific resources
- `access_grants` table: `(resource_type, resource_id, principal_type, principal_id, permission)`
- Principals: `user:{id}`, `user:*` (public), `group:{id}`
- Permissions: `read`, `write`
- Resource types: knowledge, model, prompt, tool, note, channel, file, calendar, skill
- Resolution: user is owner OR public grant exists OR direct grant OR group grant

### Admin User Privileges

- First user created (or via `WEBUI_ADMIN_EMAIL/PASSWORD`) = admin
- Admins see all data across all users (bypass tenant scoping where applicable)
- Can manage: users, groups, configs, functions, pipelines, analytics
- `BYPASS_ADMIN_ACCESS_CONTROL` (default true): admins bypass resource access checks

---

## 12. Multi-Tenant Architecture

### Tenant Resolution

Tenant (organization) resolved at the middleware level before auth:

1. **Subdomain** → `X-Organization-ID` header extraction (from reverse proxy)
2. **Header** → `X-Organization-ID` HTTP header
3. **JWT claim** → organization_id in token payload
4. **Fallback** → None (single-tenant mode)

```python
# Middleware sets request.state.organization_id
# All DB queries filter by organization_id
# All Redis keys namespaced by organization_id
```

### Data Isolation Strategy

**Database Level:**
- Every table has `organization_id` column
- All `SELECT` queries filter by `organization_id`
- All `INSERT`/`UPDATE` include `organization_id`
- Composite indexes: `(organization_id, user_id)` for performance

**Redis Namespacing:**
- Token counters: `tokens:day:{org_id}:{user_id}:{date}`
- RPM counters: `rpm:{org_id}:{user_id}`
- Memory collections: `memory-{org_id}-{user_id}` (was `user-memory-{user_id}`)

**Vector DB:**
- Qdrant and Milvus have multitenancy backends (`qdrant_multitenancy.py`, `milvus_multitenancy.py`)
- Collections scoped by organization

**File Storage:**
- Files stored with organization-scoped paths
- Access verified at download time

**AI Context:**
- Memory namespace: per-org-per-user
- Knowledge bases: per-org
- Chat history: per-org
- Tool/Function/Skill: per-org

### Organization Lifecycle

```
Create Org → Configure branding → Invite members → Assign roles
→ Members accept invitations → Users switch between orgs
→ Resources created scoped to active org → Credits consumed per org
```

---

## 13. Credit & Billing Architecture

### Credit Model

**Per-Organization Balance:**
- `organization.credit_balance` — current balance
- `organization.credit_version` — optimistic locking version

**Credit Transaction Table:**
- `credit_transaction(organization_id, user_id, amount, type, description, balance_before, balance_after)`
- Types: `usage` (deduction), `topup` (manual addition), `adjustment` (admin correction)
- Transaction history is append-only

**Usage Deduction Flow:**
1. Chat completion completes → usage reported (token counts)
2. Model pricing looked up from model config (`meta.pricing`)
3. Credit cost calculated: `(input_tokens * input_price) + (output_tokens * output_price)`
4. Optimistic lock: `UPDATE organization SET balance = balance - cost, version = version + 1 WHERE id = ? AND version = ? AND balance >= cost`
5. If lock fails or balance insufficient → error returned, chat not blocked (charges queued)
6. Credit transaction recorded

**Top-Up Flow:**
1. Admin accesses billing panel
2. Enters amount to add
3. Balance updated with optimistic locking
4. Transaction recorded with type='topup'

**Pricing Configuration:**
- Per-model pricing in `model.info.meta.pricing`:
  - `input_price_per_1m_tokens` — cost per 1M input tokens
  - `output_price_per_1m_tokens` — cost per 1M output tokens
- Default pricing when not configured
- Currency configured via `CREDIT_CURRENCY` (default: IDR)

---

## 14. AI Architecture

### Chat Pipeline (5272 lines in `utils/middleware.py`)

```
REQUEST
  │
  ▼
┌─────────────────────────────────────────────┐
│ 1. MODEL RESOLUTION                         │
│    Model lookup → arena selection → params  │
├─────────────────────────────────────────────┤
│ 2. CHAT MANAGEMENT                          │
│    New chat creation / existing chat load   │
│    Parent message resolution                │
├─────────────────────────────────────────────┤
│ 3. process_chat_payload() — Main Pipeline   │
│    ┌─────────────────────────────────────┐  │
│    │ 3a. Arena model resolution          │  │
│    │ 3b. Model params apply              │  │
│    │ 3c. Load DB messages               │  │
│    │ 3d. System prompt assembly          │  │
│    │ 3e. URL → base64 image convert      │  │
│    │ 3f. Event emitter setup             │  │
│    │ 3g. Folder project handling         │  │
│    │ 3h. Model knowledge attach          │  │
│    │ 3i. PIPELINE INLET FILTERS          │  │
│    │ 3j. FUNCTION INLET FILTERS          │  │
│    │ ┌──────────────────────────────┐    │  │
│    │ │ 3k. FEATURES PROCESSING      │    │  │
│    │ │  - Voice mode prompt         │    │  │
│    │ │  - Memory retrieval          │    │  │
│    │ │  - Web search (query→search) │    │  │
│    │ │  - Image generation setup    │    │  │
│    │ │  - Code interpreter prompt   │    │  │
│    │ └──────────────────────────────┘    │  │
│    │ 3l. Skills injection                │  │
│    │ 3m. Tool resolution (native FC)     │  │
│    │ 3n. File context extraction         │  │
│    │ 3o. Source context injection        │  │
│    │ 3p. System message merge            │  │
│    └─────────────────────────────────────┘  │
├─────────────────────────────────────────────┤
│ 4. MODEL DISPATCH                           │
│    Ollama / OpenAI / Pipeline routing       │
│    Streaming (SSE) or non-streaming         │
├─────────────────────────────────────────────┤
│ 5. RESPONSE PROCESSING                      │
│    Stream: tag detection + tool calls       │
│    Non-stream: direct extraction            │
├─────────────────────────────────────────────┤
│ 6. OUTLET FILTERS                           │
│    Pipeline outlet + Function outlet        │
├─────────────────────────────────────────────┤
│ 7. POST-PROCESSING                          │
│    Save to DB, Webhook, Background tasks    │
└─────────────────────────────────────────────┘
  │
  ▼
RESPONSE
```

### RAG Pipeline

**Document Ingestion:**
```
Upload → Load (12+ loaders) → Chunk (3 strategies)
→ Embed (4 engines) → Store (15 vector DBs)
```

**Splitting strategies:**
1. Markdown Header Splitter (splits on # through ######)
2. RecursiveCharacterTextSplitter (configurable CHUNK_SIZE/OVERLAP)
3. TokenTextSplitter (via tiktoken)

**Embedding engines:**
1. Local SentenceTransformer (default)
2. Ollama embedding API
3. OpenAI embeddings API
4. Azure OpenAI embeddings

**Vector DB backends (15):**
Chroma, Milvus, Qdrant, Pinecone, Elasticsearch, OpenSearch, PgVector, OpenGauss, MariaDB Vector, Oracle 23ai, Weaviate, S3Vector, Valkey

**Retrieval During Chat:**
```
User Message → Embed Query → Search Vector DB
→ Hybrid Search (BM25 + Vector) → Rerank (CrossEncoder/ColBERT)
→ Inject as <source> XML tags → Model sees context
```

**Web Search (30 engines):**
Generate queries → Concurrent search → Filter results → Load page content → Embed → Inject as context.

### Model Dispatch

| Provider | owned_by | API | Streaming |
|----------|---------|-----|-----------|
| Ollama | ollama | /api/chat | Native SSE |
| OpenAI-compatible | openai | /v1/chat/completions | Native SSE |
| Anthropic | (custom) | /v1/messages | SSE |
| Google | (custom) | Gemini API | SSE |
| Pipeline/Function | pipeline name | Custom | Pipeline-specific |
| Arena | arena | Random sub-model | Depends |

### Code Interpreter

Two engines:
1. **Jupyter**: Remote kernel via WebSocket, execute Python code, return results (text + images)
2. **Pyodide**: WASM-based Python in browser, no server dependency

Activation: Tag-based (`<code_interpreter>`) or native function calling (`execute_code` tool).

### Context Assembly Order

```
1. System prompt (chat settings + model config + folder settings)
2. Voice mode prompt (if voice feature enabled)
3. Memories (queried from memory DB, k=3)
4. Skills (user-selected → full content; model-attached → name+description)
5. Code interpreter prompt
6. File context (extracted from attached files)
7. Knowledge base results (RAG retrieval)
8. Web search results
9. Tool results (from tool execution loop)
→ All merged into final context for model
```

### Tools System

**13 Builtin Tool Categories (~30+ tools):**
Time, Knowledge Base, Chats, Memory, Web Search, Image Generation, Code Interpreter, Notes, Channels, Skills, Tasks, Automations, Calendar

Each can be individually disabled per-model via `model.info.meta.builtinTools`.

**Tool Execution Lifecycle:**
```
Model issues tool_call → Filter params → Execute (direct/server/MCP)
→ Process result (extract citations, handle binary) → Return to model
→ Repeat up to CHAT_RESPONSE_MAX_TOOL_CALL_ITERATIONS
```

---

## 15. Infrastructure Architecture

### Docker Architecture

**Multi-stage build:**
- Stage 1 (`build`): `node:22-alpine3.20` → `npm ci` → `npm run build` → produces `build/` (SPA static files)
- Stage 2 (`base`): `python:3.11-slim-bookworm` → installs system deps + Python deps + copies frontend build

**System packages:** git, build-essential, pandoc, gcc, netcat-openbsd, curl, jq, ffmpeg, libsm6, libxext6, zstd

**Entrypoint:** `bash start.sh`
**Port:** 8080
**Health check:** `curl /health | jq '.status == true'`

### Docker Compose (8 files)

| File | Purpose |
|------|---------|
| `docker-compose.yaml` | Main: ollama + open-webui |
| `docker-compose.gpu.yaml` | NVIDIA GPU support |
| `docker-compose.amdgpu.yaml` | AMD GPU support |
| `docker-compose.api.yaml` | Expose Ollama API port |
| `docker-compose.data.yaml` | Host data mount |
| `docker-compose.otel.yaml` | OpenTelemetry stack (Grafana LGTM) |
| `docker-compose.playwright.yaml` | Playwright browser for web scraping |
| `docker-compose.a1111-test.yaml` | Stable Diffusion (Automatic1111) |

### Storage System

**4 providers** (factory pattern via `STORAGE_PROVIDER` env var):

| Provider | Key Config | Storage |
|----------|-----------|---------|
| `local` (default) | `UPLOAD_DIR` | Local filesystem |
| `s3` | `S3_ACCESS_KEY_ID`, `S3_BUCKET_NAME`, `S3_REGION_NAME` | AWS S3 (boto3) |
| `gcs` | `GCS_BUCKET_NAME`, `GOOGLE_APPLICATION_CREDENTIALS_JSON` | Google Cloud Storage |
| `azure` | `AZURE_STORAGE_ENDPOINT`, `AZURE_STORAGE_CONTAINER_NAME` | Azure Blob Storage |

**Upload flow:** File → local temp → upload to cloud provider → path stored in DB.

### Observability

**OpenTelemetry:**
- Traces: FastAPI, SQLAlchemy, Redis, Requests, HTTPX, aiohttp instrumented
- Metrics: HTTP request count/duration, user counts (total/active/today)
- Logs: Structured JSON via loguru, forwarded via OTLP
- Pre-built stack: `docker-compose.otel.yaml` (Grafana LGTM)

**Audit Logging:**
- Levels: NONE, METADATA (user/verb/URI), REQUEST (+body), REQUEST_RESPONSE (+response)
- Always logs auth endpoints
- Configurable include/exclude path lists
- Log rotation: 10MB, compressed to zip

---

## 16. Queue & Scheduler Architecture

### Background Jobs

**Task Manager (`tasks.py`):**
- In-memory task registry (`tasks: Dict[str, asyncio.Task]`)
- Redis mirror for cross-instance coordination
- Redis pubsub channel for distributed task cancellation
- Functions: `create_task`, `stop_task`, `stop_item_tasks`, `cleanup_task`

**Scheduler Worker Loop:**
- Runs every `SCHEDULER_POLL_INTERVAL` (default 10s) + jitter
- Claims due automations from DB (limit 10, distributed `FOR UPDATE SKIP LOCKED`)
- Checks calendar events for upcoming alerts
- Sends Socket.IO + webhook notifications

**Periodic Cleanup:**
| Job | Interval | Description |
|-----|----------|-------------|
| `periodic_usage_pool_cleanup` | 3s | Reaps stale Socket.IO usage entries |
| `periodic_session_pool_cleanup` | 120s | Reaps orphaned Socket.IO sessions |
| `redis_task_command_listener` | Continuous | Cross-instance task cancellation |

### Chat Background Tasks (after response)

1. **Title generation** — generates chat title from first exchange
2. **Tag generation** — auto-tags chat based on content
3. **Follow-up generation** — suggests follow-up questions
4. **Webhook post** — notifies configured webhook URL (Slack/Discord/Teams compatible)

---

## 17. Security Documentation

### Authentication Security

- **Password hashing:** bcrypt or argon2 (configurable, argon2 preferred)
- **JWT:** HS256 with configurable secret, short-lived tokens with refresh
- **Token revocation:** Redis-backed per-token and per-user revocation
- **API keys:** UUID4-based with `sk-` prefix, optional endpoint restrictions
- **OAuth:** State parameter (CSRF protection), PKCE, Fernet-encrypted client secrets
- **Session cookies:** httpOnly, configurable SameSite, Secure flags

### Authorization Security

- **RBAC:** Role-based with granular permissions
- **Resource-level access:** Polymorphic `access_grants` table
- **Admin bypass:** Controlled via `BYPASS_ADMIN_ACCESS_CONTROL` env var
- **Models access control:** Per-model with ownership + grants
- **Knowledge access control:** Per-knowledge-base with grants

### Data Isolation

- **Organization isolation:** All tables have `organization_id`, filtered on every query
- **File isolation:** Organization-scoped storage paths
- **Redis namespacing:** All keys include `org_id`
- **Memory isolation:** Collections named `memory-{org_id}-{user_id}`
- **No global bypass flags:** `BYPASS_RETRIEVAL_ACCESS_CONTROL` removed from codebase

### Infrastructure Security

- **Security headers:** CSP, HSTS, X-Frame-Options, X-Content-Type-Options
- **CORS:** Configurable via `CORS_ALLOW_ORIGIN`
- **Compression:** Optional GZip/Brotli (disabled by default)
- **Environment:** `.env` file loaded via python-dotenv
- **Secret encryption:** OAuth client secrets encrypted with Fernet (AES-128-CBC)
- **LDAP:** Optional integration for enterprise auth

### API Security

- **Endpoint gating:** SCIM, analytics, eval arena conditional on env vars
- **API key restrictions:** Optional endpoint allowlisting
- **Rate limiting:** Sign-in rate limiter (15 attempts / 3 min per email)
- **Audit logging:** Configurable levels, always logs auth events
- **Input validation:** Pydantic models for all request bodies
- **Response sanitization:** Sensitive fields stripped from responses

---

## 18. Performance Overview

### API Performance

- **Async throughout:** All DB operations use async SQLAlchemy
- **Connection pooling:** Configurable pool size (512 default for SQLite, QueuePool for PostgreSQL)
- **Streaming:** SSE streaming for chat responses (real-time UX)
- **Database session sharing:** `DATABASE_ENABLE_SESSION_SHARING` for request-scoped sessions
- **Background processing:** Title/tag generation and webhooks run after response

### Database Optimization

- **SQLite with WAL:** Concurrent reads without blocking
- **Composite indexes:** `(organization_id, user_id)` on all tables
- **Busy timeout:** 5000ms for concurrent write contention
- **Memory-mapped I/O:** 256MB mmap for SQLite
- **Pagination:** `limit/offset` on list endpoints
- **Idempotent migrations:** Safe to re-run

### Caching

- **Model cache:** `app.state.MODELS` in-memory dictionary
- **Tool/Function cache:** Content hash-based invalidation
- **Config cache:** DB-backed with runtime updates
- **Redis:** Key-value store for sessions, rate limiting, task coordination
- **No application-level response caching** — chat responses are dynamic

### Frontend Performance

- **SPA architecture:** Single page load, client-side routing
- **Code splitting:** SvelteKit automatic chunk splitting
- **Lazy loading:** Components loaded on demand via SvelteKit routing
- **Virtual scrolling:** Not implemented — pagination used for chat list
- **Streaming:** SSE for real-time token display (fluid UX with delta chunking)
- **Image optimization:** Base64 → URL conversion for chat response images

---

## 19. Dependency Mapping

### Frontend → Backend Dependencies

```
┌─────────────────────────────────────────────────┐
│ FRONTEND (SvelteKit SPA)                         │
│                                                   │
│ Routes ───→ Stores ───→ API Clients             │
│    │           │            │                     │
│    │           │            ├── auths/ → /auths  │
│    │           │            ├── chats/ → /chats  │
│    │           │            ├── models/ → /models│
│    │           │            ├── knowledge/ → /k..│
│    │           │            └── ... (28 more)     │
│    │           │                                  │
│    │           ├── user store ← /auths/session   │
│    │           ├── config store ← /configs       │
│    │           ├── models store ← /models        │
│    │           └── chats store ← /chats          │
│    │                                              │
│    └── Socket.IO Client ← /ws/socket.io          │
└──────────────────────┬──────────────────────────┘
                       │ HTTP + WebSocket
                       ▼
┌─────────────────────────────────────────────────┐
│ BACKEND (FastAPI)                                │
│                                                   │
│ Middleware → Routers → Models → Database         │
│     │          │         │         │               │
│     │          │         │         ├── SQLite     │
│     │          │         │         └── PostgreSQL │
│     │          │         │                        │
│     │          │         ├── Redis                │
│     │          │         ├── Vector DB            │
│     │          │         └── Storage Provider     │
│     │          │                                  │
│     │          ├── auths ↔ Auth Tables            │
│     │          ├── chats ↔ Chat Pipeline          │
│     │          ├── models ↔ Model Table           │
│     │          └── ... (28 more)                  │
└─────────────────────────────────────────────────┘
```

### Backend Internal Dependency Map

```
main.py
├── config.py ← env.py ← .env
├── internal/db.py (sync + async engines)
├── internal/config.py (DB-hosted config)
├── routers/* (30 files)
│   ├── → models/* (25 files)
│   │   ├── → internal/db.py (async engine)
│   │   └── → utils/tenant_db.py (tenant/workspace filters)
│   ├── → utils/auth.py (JWT, OAuth, API keys)
│   ├── → utils/middleware.py (chat pipeline)
│   ├── → utils/rate_limit.py (sign-in limiter)
│   ├── → utils/redis.py (redis client)
│   ├── → utils/webhook.py (webhook posts)
│   ├── → utils/tools.py (tool loading)
│   ├── → utils/plugin.py (function loading)
│   └── → retrieval/* (vectors, search, loaders)
├── socket/main.py (Socket.IO handlers)
├── tasks.py (background task manager)
└── utils/
    ├── asgi_middleware.py (pure ASGI middlewares)
    ├── security_headers.py
    ├── audit.py
    ├── task.py (automation/calendar scheduler)
    └── ... (25+ utility modules)
```

### Infrastructure Dependency Map

```
open-webui container
├── FastAPI app (port 8080)
├── Redis connection (optional)
├── Database connection (SQLite/PostgreSQL)
├── Vector DB connection (15 backends)
├── Storage provider (local/S3/GCS/Azure)
└── External integrations
    ├── Ollama server (optional)
    ├── OpenAI-compatible APIs
    ├── Web search engines (30)
    ├── OAuth providers
    └── OpenTelemetry collector
```

---

## 20. Configuration Guide

### Configuration Hierarchy

1. **Environment variables** (system `os.environ`)
2. **`.env` file** (project root, loaded via `python-dotenv`)
3. **Code defaults** (fallback values in `os.getenv('VAR', 'default')`)
4. **Database configs** (persistent config stored in `config` table, managed at runtime)

### Major Environment Variable Categories

| Category | Count (approx) | Key Variables |
|----------|---------------|---------------|
| Logging/Env | 10 | `GLOBAL_LOG_LEVEL`, `ENV`, `DOCKER` |
| Database | 20 | `DATABASE_URL`, `DATABASE_POOL_SIZE`, SQLite PRAGMAs |
| Redis | 10 | `REDIS_URL`, `REDIS_KEY_PREFIX` |
| WebSocket | 15 | `ENABLE_WEBSOCKET_SUPPORT`, `WEBSOCKET_MANAGER` |
| Auth/Secrets | 15 | `WEBUI_SECRET_KEY`, `JWT_EXPIRES_IN`, admin auto-create |
| OAuth | 10 | `OAUTH_PROVIDERS`, `GOOGLE_CLIENT_ID`, etc. |
| Feature Flags | 15 | `SAFE_MODE`, `ENABLE_SIGNUP`, `ENABLE_CHANNELS` |
| AI Providers | 40+ | `OPENAI_API_BASE_URLS`, `OLLAMA_BASE_URLS`, GEMINI key |
| RAG/Retrieval | 30+ | `RAG_EMBEDDING_ENGINE`, `CHUNK_SIZE`, `RAG_TOP_K` |
| Web Search | 25+ | `WEB_SEARCH_ENGINE`, 25 provider API keys |
| Storage | 10+ | `STORAGE_PROVIDER`, `S3_*`, `GCS_*` |
| Observability | 15+ | `ENABLE_OTEL`, `OTEL_*`, `AUDIT_LOG_LEVEL` |
| UI | 10+ | `WEBUI_NAME`, `DEFAULT_LOCALE`, `WEBUI_BANNERS` |

**Total: ~200+ environment variables**

### Persistent Config System

`config.py` defines 250+ `ConfigVar` entries stored in the database `config` table. These are read/written at runtime via the `/api/v1/configs` API. ConfigVars include:
- OAuth provider configurations
- Model configurations
- RAG settings
- Audio STT/TTS settings
- Image generation settings
- Web search engine configurations
- Tool server configurations

### Feature Flags

Major feature flags (environment variables):
- `ENABLE_SIGNUP` — allow self-registration
- `ENABLE_OAUTH_SIGNUP` — auto-create accounts via OAuth
- `ENABLE_MULTI_ORG` — enable multi-tenant features
- `ENABLE_CHANNELS` — enable real-time channels
- `ENABLE_CODE_EXECUTION` — enable code interpreter
- `ENABLE_WEBSOCKET` — enable Socket.IO real-time
- `ENABLE_AUTOMATIONS` — enable automation scheduler
- `ENABLE_CALENDAR` — enable calendar features
- `ENABLE_ADMIN_ANALYTICS` — enable analytics pages
- `ENABLE_SCIM` — enable SCIM 2.0 provisioning
- `ENABLE_OTEL` — enable OpenTelemetry
- `ENABLE_COMPRESSION_MIDDLEWARE` — GZip/Brotli

---

## 21. Deployment Guide

### Local Development

```bash
# Clone
git clone --depth 1 https://github.com/open-webui/open-webui.git
cd open-webui

# Python venv
uv venv --python 3.11
source .venv/bin/activate
NODE_OPTIONS="--max-old-space-size=8192" uv pip install -e .

# Environment
cp .env.example .env
# Edit .env: set WEBUI_SECRET_KEY, model providers, etc.

# Run
cd backend
uvicorn open_webui.main:app --port 8080 --host 0.0.0.0
```

### Docker Deployment

```bash
# Basic
docker compose up -d

# With GPU
docker compose -f docker-compose.yaml -f docker-compose.gpu.yaml up -d

# With monitoring
docker compose -f docker-compose.yaml -f docker-compose.otel.yaml up -d

# With browser automation
docker compose -f docker-compose.yaml -f docker-compose.playwright.yaml up -d
```

### Build Process

1. **Frontend:** `npm run build` → SvelteKit compiles to `build/` directory (10-15 min)
2. **Backend:** Python package install + Alembic migrations on startup
3. **Static files:** `build/` syncs to `backend/open_webui/static/` by `config.py`
4. **Health check:** `/ready` returns 200 when startup complete + DB/Redis ready

### Scaling Strategy

- **Vertical:** Increase Uvicorn workers (`UVICORN_WORKERS` env var)
- **Horizontal:** Multiple instances behind reverse proxy (Nginx), Redis required for:
  - Session sharing (StarSessions with RedisStore)
  - WebSocket coordination (Redis Socket.IO manager)
  - Task coordination (Redis pubsub)
  - Rate limiting
- **Database:** PostgreSQL recommended for multi-instance deployments (SQLite is single-writer)
- **Vector DB:** External vector DB (Qdrant, Milvus, Pinecone) for production RAG
- **File Storage:** Object storage (S3/GCS/Azure) for production file persistence

---

## 22. Data Flow Documentation

### Chat Completion Data Flow

```
User types message → Svelte component → POST /api/chat/completions
    →
Middleware chain processes request
    →
Router extracts form_data + user + model
    →
Chat created/saved to DB (dual-write: chat + chat_message)
    →
process_chat_payload():
  → Load messages from DB
  → Apply system prompt
  → Run inlet filters (pipeline + function)
  → Process features: memory, web search, image gen, code interpreter
  → Inject skills, tools, file context
  → Query RAG pipeline (knowledge + web search)
  → Assemble final context
    →
Model dispatch (Ollama/OpenAI/Pipeline)
    →
Streaming response (SSE chunks)
    →
process_chat_response():
  → Parse stream for tags (think, code_interpreter, tool_calls)
  → Emit events to Socket.IO frontend
  → Real-time save to DB (ENABLE_REALTIME_CHAT_SAVE)
  → Run outlet filters
  → Run background tasks (title, tags, follow-ups)
  → Post webhook
    →
Frontend receives SSE events → Updates chat UI in real-time
```

### RAG Ingestion Data Flow

```
User uploads file → POST /api/v1/files/upload
    →
File stored (local/S3/GCS/Azure)
    →
File attached to knowledge base → POST /api/v1/knowledge/{id}/file/add
    →
Process file → POST /api/v1/retrieval/process/file
    →
Load document (PDF/DOCX/CSV/HTML/YouTube via loaders)
    →
Split into chunks (markdown header / character / token splitter)
    →
Embed chunks (sentence-transformers / Ollama / OpenAI)
    →
Store in vector DB (Chroma / Qdrant / Milvus / ...)
    →
Ready for retrieval during chat
```

### Web Search Data Flow

```
User enables web search → During chat pipeline
    →
Generate search queries from user message + conversation history
    →
Concurrent search across 30 engine backends
    →
Filter results (domain allowlist/blocklist)
    →
Load page content via web loaders
    →
Embed and store in temporary vector collection
    →
Query alongside knowledge bases during RAG retrieval
    →
Inject results as <source> XML tags in context
```

### Real-Time Channel Data Flow

```
User opens channel → Svelte page loads → Socket.IO join-channel
    →
User types message → POST /api/v1/channels/{id}/messages
    →
Message saved to message table
    →
Socket.IO emits events:channel to all room members
    →
Other users' frontends receive event → Update chat UI
    →
Presence: heartbeat every 30s → update last_seen_at
    →
Channel CRUD (create, update, delete, pin, mute) all real-time broadcast
```

---

## 23. Component Relationship Documentation

### Frontend Component Hierarchy

```
+layout.svelte (root)
├── Toaster
├── Socket.IO setup (invisible)
└── <slot>
    ├── auth/+page.svelte (public)
    │   └── SignIn/SignUp forms
    │
    └── (app)/+layout.svelte
        ├── SettingsModal
        ├── ChangelogModal
        ├── UpdateInfoToast
        ├── Sidebar (1659 lines)
        │   ├── UserMenu
        │   ├── SearchInput + SearchModal
        │   ├── ChatItem (per chat)
        │   │   └── ChatMenu (rename/delete/pin/archive)
        │   ├── Folders + FolderModal + FolderMenu
        │   │   └── RecursiveFolder
        │   ├── PinnedModelList → PinnedModelItem
        │   ├── ChannelItem + ChannelModal
        │   └── UserStatusModal
        │
        └── <slot>
            ├── +page.svelte (main chat)
            │   ├── Chat.svelte
            │   │   ├── Messages (chat history)
            │   │   │   ├── Message (per message)
            │   │   │   │   ├── ContentRenderer (+ FloatingButtons)
            │   │   │   │   ├── CodeBlock (highlight.js/shiki)
            │   │   │   │   ├── KaTeX/Mermaid/Charts
            │   │   │   │   └── Citations, Sources, Usage
            │   │   │   └── ...
            │   │   ├── MessageInput
            │   │   │   ├── InputMenu (Chats/Files/Knowledge/Notes)
            │   │   │   ├── FilesOverlay + CallOverlay
            │   │   │   ├── CommandSuggestionList
            │   │   │   │   ├── Commands/Models
            │   │   │   │   ├── Commands/Prompts
            │   │   │   │   ├── Commands/Knowledge
            │   │   │   │   ├── Commands/Skills
            │   │   │   │   └── Commands/Emojis
            │   │   │   ├── VoiceRecording
            │   │   │   └── TerminalMenu
            │   │   ├── ModelSelector
            │   │   ├── Settings (chat-level)
            │   │   │   ├── Controls/Controls + Valves
            │   │   │   └── Embeds
            │   │   ├── FileNav (with sub-components)
            │   │   │   ├── FileEntryRow, FilePreview
            │   │   │   ├── FileCodeEditor, NotebookView
            │   │   │   ├── SqliteView, JsonTreeView
            │   │   │   └── PortList, PortPreview
            │   │   ├── Artifacts
            │   │   ├── ChatControls (overview, activity)
            │   │   └── ChatPlaceholder
            │   └── ...
            │
            ├── admin/+layout.svelte
            │   ├── Users.svelte
            │   │   ├── UserList → UserItem
            │   │   │   ├── AddUserModal, EditUserModal
            │   │   │   └── UserChatsModal
            │   │   └── Groups.svelte
            │   │       ├── GroupItem, GroupPreviewPanel
            │   │       ├── EditGroupModal
            │   │       ├── General, Users, Permissions tabs
            │   │       └── Permissions.svelte (1015 lines)
            │   ├── Settings.svelte
            │   │   ├── General, Interface, Models, Connections
            │   │   ├── Documents, WebSearch, Images, Audio
            │   │   ├── Database, Integrations, CodeExecution
            │   │   ├── Evaluations (ArenaModelModal)
            │   │   └── Pipelines
            │   ├── Analytics.svelte + sub-components
            │   ├── Evaluations.svelte + sub-components
            │   └── Functions.svelte + sub-components
            │
            ├── workspace/* (CRUD pages)
            │   ├── Models.svelte + sub-components
            │   ├── Knowledge.svelte + sub-components
            │   ├── Prompts.svelte + sub-components
            │   ├── Tools.svelte + sub-components
            │   └── Skills.svelte + sub-components
            │
            ├── channels/[id]/+page.svelte
            │   ├── Channel.svelte
            │   │   ├── Navbar → ChannelInfoModal
            │   │   ├── Messages → Message → ProfilePreview
            │   │   ├── MessageInput → InputMenu + MentionList
            │   │   ├── Thread
            │   │   ├── PinnedMessagesModal
            │   │   └── WebhooksModal → WebhookItem
            │   └── ...
            │
            ├── notes/* (Notes feature)
            ├── calendar/* (Calendar feature)
            ├── automations/* (Automations feature)
            └── playground/* (Playground feature)
```

### Backend Component Dependencies

```
main.py (3029 lines)
├── config.py (4047) — persistent config loading
│   └── env.py (1069) — environment variable definitions
├── internal/db.py — dual-engine setup
├── routers/* (30 files)
│   ├── auths.py ← utils/auth.py (JWT, OAuth, API keys)
│   ├── chats.py ← utils/middleware.py (chat pipeline)
│   ├── retrieval.py ← retrieval/* (vectors, loaders, web search)
│   ├── models.py ← utils/models.py
│   ├── users.py ← utils/auth.py
│   └── ...
├── models/* (25 files) — SQLAlchemy table definitions
│   ├── ← internal/db.py (async engine)
│   └── ← utils/tenant_db.py (tenant/workspace filters)
├── utils/
│   ├── middleware.py (5272) — chat processing pipeline
│   │   ├── → utils/embeddings.py (embedding models)
│   │   ├── → utils/tools.py (tool resolution)
│   │   ├── → utils/plugin.py (function loading)
│   │   ├── → utils/code_interpreter.py (Jupyter/Pyodide)
│   │   ├── → utils/filter.py (inlet/outlet filters)
│   │   └── → utils/payload.py (message assembly)
│   ├── asgi_middleware.py — Redirect, CommitSession, AuthToken, WS Guard
│   ├── auth.py — JWT, OAuth, API key auth
│   ├── rate_limit.py — Sign-in rate limiter
│   ├── redis.py — Redis client factory
│   ├── audit.py — Audit logging middleware
│   ├── security_headers.py — Security headers
│   ├── oauth.py — OAuth manager
│   └── ...
├── retrieval/
│   ├── utils.py (1567) — RAG pipeline orchestration
│   ├── vector/main.py — Vector DB factory
│   │   └── vector/dbs/* (15 backends)
│   ├── web/main.py — Web search dispatcher
│   │   └── web/* (30 search engines)
│   ├── loaders/main.py — Document loaders
│   │   └── loaders/* (12+ loaders)
│   └── vector/factory.py — Vector DB client factory
├── socket/main.py — Socket.IO event handlers
│   └── ← utils/redis.py (Redis manager)
├── tasks.py — Background task manager
│   └── ← utils/redis.py (pubsub)
└── storage/provider.py — Storage providers (4)
```

---

## Appendix: Key Statistics

| Metric | Value |
|--------|-------|
| Backend Python files | ~130+ |
| Frontend Svelte/TS files | ~600+ |
| Total lines of code | ~120,000+ |
| API routers | 30 |
| Total endpoints | 436 |
| Database tables | 37 |
| Model files | 25 |
| Alembic migrations | 44 |
| Middleware layers | 10 |
| Frontend routes | 46 +page.svelte |
| Writable stores | 55 |
| API client modules | 30 |
| Component files | ~530+ |
| Chat pipeline (middleware.py) | 5,272 lines |
| Main app (main.py) | 3,029 lines |
| Config system (config.py) | 4,047 lines |
| Environment variables | ~200+ |
| i18n locales | 63 |
| Web search engines | 30 |
| Vector DB backends | 15 |
| Embedding engines | 4 |
| Document loaders | 12+ |
| Builtin tool categories | 13 |
| Supported OAuth providers | Google, Microsoft, GitHub + custom |
| Storage providers | 4 (local, S3, GCS, Azure) |
| Docker compose files | 8 |
| Python dependencies | 261 packages |

---

*Documentation generated from comprehensive codebase analysis — June 2026*
*StratLogic Open WebUI v0.9.6 — Branch: stratlogic-product*
