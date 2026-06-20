# StratLogic Open WebUI — AI Agent Context

> For AI coding agents (Antigravity, Claude Code, Cursor, Copilot) working on this project.

## Project Identity

- **Product:** Multi-tenant SaaS AI chat platform
- **Fork:** Open WebUI v0.9.6 (stratlogic-product branch)
- **Stack:** FastAPI backend + SvelteKit SPA frontend
- **Database:** SQLite (dev) / PostgreSQL (prod), SQLAlchemy async
- **Key files:** `backend/open_webui/main.py` (3029L), `src/lib/stores/index.ts` (55 stores)

## Architecture Docs

Read these BEFORE making changes:

| File | Content |
|------|---------|
| `docs/architecture/00-comprehensive-documentation.md` | Full codebase analysis — 37 tables, 30 routers, 55 stores |
| `docs/architecture/01-architecture-decisions.md` | All 37 architecture decisions (business model, RBAC, billing, isolation) |
| `docs/architecture/02-implementation-plan.md` | Step-by-step implementation guide |

## Conventions

- **Python:** Type hints required. Async SQLAlchemy for all runtime DB. Use `patch()` tool for targeted edits, never bulk Python scripts.
- **Frontend:** Svelte 5 runes mode (`$props()`, not `export let`). SPA (ssr=false). Use `workspaceHeaders()` interceptor pattern.
- **Build:** Run `NODE_OPTIONS=--max-old-space-size=8192 npx vite build --logLevel error` (12 min). Build AFTER every file change, never batch.
- **Migrations:** Alembic. Auto-run on startup. All migrations must be idempotent.
- **Secrets:** NEVER write .env via terminal() or write_file() — use execute_code with Python open().write().

## Critical Pitfalls

1. `read_file()` output has line-number prefixes (`1|...`). NEVER write back raw.
2. `process.wait` timeout clamped to 180s. For long builds, poll in loop.
3. Sidebar.svelte is 1659 lines. Use targeted single-line patch(), never multi-line replace.
4. `search_files` may return stale results for large files. Verify with `grep -n` in terminal.
5. Static files go in `build/static/`, NOT `backend/open_webui/static/` (config.py syncs on startup).
6. Alembic DB version may be stale. Check `alembic current` before running migrations.
