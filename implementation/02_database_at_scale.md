# Implementation Plan: Database at Scale
**Antigravity IDE — Open WebUI SaaS Layer**
**Document Version:** 1.0
**Focus Area:** PostgreSQL migration, connection pooling, read replicas, caching, migrations

---

## Overview

Open WebUI defaults to SQLite, which is completely unsuitable for multi-user SaaS workloads. This plan covers the full database scaling path: migrating to PostgreSQL, introducing connection pooling, adding read replicas for query distribution, caching frequently-accessed data in Redis, and establishing a safe migration workflow using Alembic.

---

## Current State (Open WebUI Baseline)

- SQLite via SQLAlchemy ORM (`backend/open_webui/internal/db.py`)
- No connection pooling — SQLite doesn't need it
- No caching layer
- Schema managed by Peewee migrations in older versions, now SQLAlchemy
- All reads and writes go to a single file

---

## Target Architecture

```
Application Layer (multiple FastAPI workers)
          │
          ▼
    PgBouncer (connection pooler — port 5432)
     │              │
     ▼              ▼
PostgreSQL       PostgreSQL
Primary          Read Replica(s)
(writes)         (reads/analytics)
          │
          ▼
      Redis (cache + session store)
```

---

## Phase 1: PostgreSQL Migration

### 1.1 Switch Database URL

In `backend/open_webui/env.py`, update:
```python
DATABASE_URL = os.environ.get(
    "DATABASE_URL",
    "postgresql+asyncpg://owui:password@localhost:5432/openwebui"
)
```

Remove the SQLite fallback. Make `DATABASE_URL` required in production.

### 1.2 Replace SQLAlchemy Engine Config

Update `backend/open_webui/internal/db.py`:
```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

engine = create_async_engine(
    DATABASE_URL,
    pool_size=20,           # persistent connections kept open
    max_overflow=10,        # extra connections beyond pool_size
    pool_timeout=30,        # seconds to wait for a connection
    pool_recycle=1800,      # recycle connections every 30 min (avoids stale)
    pool_pre_ping=True,     # test connection before use
    echo=False,             # set True for SQL debug logging
)

AsyncSessionLocal = sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,
)

async def get_db():
    async with AsyncSessionLocal() as session:
        yield session
```

### 1.3 Update All ORM Models

Ensure all models use PostgreSQL-compatible types:
- Replace `Text` with `String` or `Text` (both work in PG)
- Replace any SQLite-specific JSON handling with `JSONB` for indexable JSON:

```python
from sqlalchemy.dialects.postgresql import JSONB, UUID as PG_UUID
import uuid

class Chat(Base):
    __tablename__ = "chats"
    id           = Column(PG_UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    organization_id = Column(PG_UUID(as_uuid=True), ForeignKey("organizations.id"), index=True)
    user_id      = Column(PG_UUID(as_uuid=True), ForeignKey("users.id"), index=True)
    title        = Column(String(500))
    chat         = Column(JSONB)               # replaces Text JSON blobs
    created_at   = Column(DateTime, server_default=func.now())
    updated_at   = Column(DateTime, onupdate=func.now())
```

### 1.4 Add Database Indexes

Critical indexes for SaaS query patterns:
```sql
-- Most frequent: load user's chats
CREATE INDEX idx_chats_user_id          ON chats(user_id);
CREATE INDEX idx_chats_org_id           ON chats(organization_id);
CREATE INDEX idx_chats_updated_at       ON chats(updated_at DESC);

-- Composite: org + user (used in permission checks)
CREATE INDEX idx_chats_org_user         ON chats(organization_id, user_id);

-- Full-text search on chat title
CREATE INDEX idx_chats_title_fts        ON chats USING gin(to_tsvector('english', title));

-- JSONB field indexing for chat content queries
CREATE INDEX idx_chats_content_gin      ON chats USING gin(chat);

-- Auth lookups
CREATE INDEX idx_users_email            ON users(email);
CREATE INDEX idx_users_org              ON users(current_organization_id);
CREATE UNIQUE INDEX idx_users_sso       ON users(sso_provider, sso_subject) WHERE sso_provider IS NOT NULL;

-- Rate limiting (see rate limiting plan)
CREATE INDEX idx_usage_logs_user_ts     ON usage_logs(user_id, created_at DESC);
```

---

## Phase 2: Connection Pooling with PgBouncer

### 2.1 Why PgBouncer

PostgreSQL creates a separate process per connection — expensive. With 10 FastAPI workers × 20 pool connections = 200 concurrent PG connections. At scale this crushes PG. PgBouncer acts as a proxy, multiplexing thousands of app connections into a small pool of real PG connections.

### 2.2 PgBouncer Configuration

`pgbouncer.ini`:
```ini
[databases]
openwebui = host=postgres-primary port=5432 dbname=openwebui

[pgbouncer]
listen_port        = 5432
listen_addr        = 0.0.0.0
auth_type          = scram-sha-256
auth_file          = /etc/pgbouncer/userlist.txt
pool_mode          = transaction        ; best for async workloads
max_client_conn    = 1000              ; max connections from app
default_pool_size  = 25               ; connections to real PG per db/user
min_pool_size      = 5
reserve_pool_size  = 5
reserve_pool_timeout = 3
server_idle_timeout = 600
log_connections    = 0
log_disconnections = 0
```

### 2.3 Docker Compose Entry

```yaml
pgbouncer:
  image: bitnami/pgbouncer:latest
  environment:
    POSTGRESQL_HOST: postgres-primary
    POSTGRESQL_PORT: 5432
    POSTGRESQL_DATABASE: openwebui
    POSTGRESQL_USERNAME: owui
    POSTGRESQL_PASSWORD: ${DB_PASSWORD}
    PGBOUNCER_POOL_MODE: transaction
    PGBOUNCER_MAX_CLIENT_CONN: 1000
    PGBOUNCER_DEFAULT_POOL_SIZE: 25
  ports:
    - "5432:5432"
  depends_on:
    - postgres-primary
```

App connects to `pgbouncer:5432` instead of `postgres-primary:5432` directly.

---

## Phase 3: Read Replicas

### 3.1 Read/Write Splitting

Create a dual-engine setup in `db.py`:

```python
write_engine = create_async_engine(PRIMARY_DATABASE_URL, ...)
read_engine  = create_async_engine(REPLICA_DATABASE_URL, ...)

WriteSession = sessionmaker(write_engine, class_=AsyncSession, ...)
ReadSession  = sessionmaker(read_engine,  class_=AsyncSession, ...)

async def get_write_db():
    async with WriteSession() as session:
        yield session

async def get_read_db():
    async with ReadSession() as session:
        yield session
```

Use `get_read_db` for all `GET` endpoints (chat list, search, user profile).
Use `get_write_db` for all `POST/PUT/DELETE` endpoints.

### 3.2 Multiple Replicas with Round-Robin

For 2+ replicas, implement round-robin selection:
```python
REPLICA_URLS = os.environ.get("REPLICA_DATABASE_URLS", "").split(",")

replica_engines = [create_async_engine(url, ...) for url in REPLICA_URLS]
replica_sessions = [sessionmaker(e, class_=AsyncSession) for e in replica_engines]
_replica_idx = 0

async def get_read_db():
    global _replica_idx
    session_factory = replica_sessions[_replica_idx % len(replica_sessions)]
    _replica_idx += 1
    async with session_factory() as session:
        yield session
```

### 3.3 PostgreSQL Streaming Replication Setup

In `postgresql.conf` (primary):
```
wal_level           = replica
max_wal_senders     = 5
wal_keep_size       = 1GB
synchronous_commit  = on
```

In `pg_hba.conf` (primary):
```
host replication replicator replica_ip/32 scram-sha-256
```

On replica server:
```bash
pg_basebackup -h primary_ip -U replicator -D /var/lib/postgresql/data -Fp -Xs -P -R
```

---

## Phase 4: Redis Caching Layer

### 4.1 What to Cache

| Data | TTL | Reason |
|---|---|---|
| User session / JWT validation | 15 min | Avoid DB lookup every request |
| User profile | 5 min | Frequently read, rarely written |
| Organization config / settings | 10 min | Read on every request |
| Model list per org | 2 min | Shared across all users in org |
| Rate limit counters | 1 min window | Must be fast, shared across workers |
| Chat metadata (not content) | 30 sec | Sidebar list refresh |

### 4.2 Redis Client Setup

Install: `redis[hiredis]`, `aiocache`

Create `utils/cache.py`:
```python
import redis.asyncio as aioredis
import json

redis_client = aioredis.from_url(
    os.environ.get("REDIS_URL", "redis://localhost:6379"),
    encoding="utf-8",
    decode_responses=True,
    max_connections=50,
)

async def cache_get(key: str):
    value = await redis_client.get(key)
    return json.loads(value) if value else None

async def cache_set(key: str, value, ttl: int = 300):
    await redis_client.setex(key, ttl, json.dumps(value))

async def cache_del(key: str):
    await redis_client.delete(key)

async def cache_del_prefix(prefix: str):
    keys = await redis_client.keys(f"{prefix}*")
    if keys:
        await redis_client.delete(*keys)
```

### 4.3 Cache Decorator

```python
def cached(ttl=300, key_fn=None):
    def decorator(func):
        async def wrapper(*args, **kwargs):
            key = key_fn(*args, **kwargs) if key_fn else f"{func.__name__}:{args}:{kwargs}"
            cached_val = await cache_get(key)
            if cached_val is not None:
                return cached_val
            result = await func(*args, **kwargs)
            await cache_set(key, result, ttl)
            return result
        return wrapper
    return decorator

# Usage
@cached(ttl=600, key_fn=lambda org_id: f"org:config:{org_id}")
async def get_org_config(org_id: str):
    return await db.query(Organization).filter_by(id=org_id).first()
```

### 4.4 Cache Invalidation Strategy

Invalidate on mutations:
```python
# In organization update endpoint
await update_organization(org_id, data)
await cache_del(f"org:config:{org_id}")
await cache_del_prefix(f"org:members:{org_id}:")
```

---

## Phase 5: Alembic Migration Workflow

### 5.1 Setup

```bash
pip install alembic
alembic init alembic
```

Configure `alembic/env.py`:
```python
from open_webui.internal.db import Base
from open_webui.models import *   # import all models so Alembic sees them

target_metadata = Base.metadata
config.set_main_option("sqlalchemy.url", DATABASE_URL)
```

### 5.2 Migration Commands

```bash
# Generate migration from model changes
alembic revision --autogenerate -m "add_organizations_table"

# Apply all pending migrations
alembic upgrade head

# Rollback one step
alembic downgrade -1

# Show history
alembic history
```

### 5.3 Safe Production Migration Process

1. Run `alembic upgrade head` in a staging environment first
2. Take a `pg_dump` snapshot before production migration
3. Run with `--sql` flag to review generated SQL before applying
4. Apply during low-traffic window
5. Monitor error rates for 15 minutes post-migration
6. Rollback script ready: `alembic downgrade -1`

### 5.4 Zero-Downtime Migration Rules

- **Always additive first**: add columns as nullable, backfill, then add constraints
- **Never rename** columns — add new, migrate, drop old in separate deploys
- **Never drop** columns until all app instances no longer reference them
- Use `op.execute()` for bulk backfills in migration scripts

Example safe migration:
```python
def upgrade():
    # Step 1: add nullable column
    op.add_column('users', sa.Column('organization_id', PG_UUID(), nullable=True))
    # Step 2: backfill (in migration script)
    op.execute("UPDATE users SET organization_id = (SELECT id FROM organizations LIMIT 1)")
    # Step 3: add index (CONCURRENTLY — no table lock)
    op.execute("CREATE INDEX CONCURRENTLY idx_users_org ON users(organization_id)")
    # Step 4: add NOT NULL in a SEPARATE future migration after app is updated
```

---

## Phase 6: Monitoring & Observability

### 6.1 Key Database Metrics to Track

- Query latency (p50, p95, p99) — alert if p99 > 500ms
- Connection pool utilization — alert if > 80%
- Replication lag — alert if > 5 seconds
- Cache hit rate — target > 80%
- Slow query log — log any query > 100ms

### 6.2 Tooling

| Tool | Purpose |
|---|---|
| `pg_stat_statements` | Identify slow queries |
| `pgBadger` | Analyze PG logs |
| Prometheus `postgres_exporter` | Metrics to Grafana |
| Redis `INFO` endpoint | Cache hit/miss rates |
| Sentry | Application-level DB errors |

### 6.3 Enable pg_stat_statements

```sql
-- In postgresql.conf
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all

-- After restart
CREATE EXTENSION pg_stat_statements;

-- Query to find top slow queries
SELECT query, mean_exec_time, calls
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;
```

---

## Infrastructure as Code (Docker Compose — Development)

```yaml
services:
  postgres-primary:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: openwebui
      POSTGRES_USER: owui
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./postgres/primary.conf:/etc/postgresql/postgresql.conf
    command: postgres -c config_file=/etc/postgresql/postgresql.conf

  postgres-replica:
    image: postgres:16-alpine
    environment:
      PGUSER: replicator
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    depends_on:
      - postgres-primary

  pgbouncer:
    image: bitnami/pgbouncer:latest
    depends_on:
      - postgres-primary

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 512mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

---

## Environment Variables Reference

```bash
# Primary database (writes)
DATABASE_URL=postgresql+asyncpg://owui:password@pgbouncer:5432/openwebui

# Read replicas (reads) — comma-separated
REPLICA_DATABASE_URLS=postgresql+asyncpg://owui:password@replica1:5432/openwebui,postgresql+asyncpg://owui:password@replica2:5432/openwebui

# Redis
REDIS_URL=redis://redis:6379/0
REDIS_MAX_CONNECTIONS=50

# Pool settings
DB_POOL_SIZE=20
DB_MAX_OVERFLOW=10
DB_POOL_TIMEOUT=30
```

---

## Migration Checklist (SQLite → PostgreSQL)

- [ ] Export existing SQLite data with `sqlite3 .dump`
- [ ] Convert SQLite dump to PostgreSQL-compatible SQL (use `pgloader`)
- [ ] Import into dev PostgreSQL and validate row counts
- [ ] Run full test suite against PostgreSQL
- [ ] Update all remaining `Text`-based JSON blobs to `JSONB`
- [ ] Add all indexes defined in Phase 1.4
- [ ] Verify Alembic baseline matches current schema
- [ ] Deploy PgBouncer and validate connection limits
- [ ] Redis deployed and cache hit rate > 70% in staging
- [ ] Replica streaming replication lag < 1 second
- [ ] Monitoring dashboards live
