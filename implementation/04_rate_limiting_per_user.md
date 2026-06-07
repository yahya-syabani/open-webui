# Implementation Plan: Rate Limiting Per User
**Antigravity IDE — Open WebUI SaaS Layer**
**Document Version:** 1.0
**Focus Area:** Per-user rate limiting, token quotas, plan-based throttling, abuse prevention

---

## Overview

Without rate limiting, a single power user or abusive client can saturate your GPU nodes and degrade the experience for all other users. This plan covers a multi-layered rate limiting system: request-per-minute limits at the API gateway level, token-per-day/month quotas at the LiteLLM layer, and a Redis-backed sliding window counter in Open WebUI's own middleware for fine-grained control.

---

## Current State (Open WebUI Baseline)

- No rate limiting of any kind
- No per-user token tracking
- No quota enforcement
- All users have equal access to all models
- No abuse detection

---

## Target Architecture

```
Request → Nginx (IP-level throttle)
              │
              ▼
       Open WebUI Middleware (per-user RPM sliding window)
              │
              ▼
       LiteLLM Proxy (per-key TPM/RPM + budget limits)
              │
              ▼
       LLM Backend (Ollama / OpenAI)
```

Three layers, each catching different abuse patterns:
- **Nginx**: raw IP flood protection
- **Open WebUI Middleware**: per-user request rate (RPM)
- **LiteLLM**: per-user token rate (TPM) and monthly spend budget

---

## Phase 1: Plan Definitions

### 1.1 Rate Limit Tiers

Define plan limits as a central config in `backend/open_webui/config/plans.py`:

```python
from dataclasses import dataclass
from typing import Optional

@dataclass
class PlanLimits:
    plan:                  str
    rpm:                   int          # requests per minute
    tpm:                   int          # tokens per minute
    daily_token_limit:     int          # tokens per day
    monthly_token_limit:   int          # tokens per month
    monthly_budget_usd:    float        # max USD spend / month
    max_context_length:    int          # max tokens in a single request
    concurrent_requests:   int          # simultaneous in-flight requests
    models_allowed:        list[str]

PLAN_LIMITS: dict[str, PlanLimits] = {
    "free": PlanLimits(
        plan                = "free",
        rpm                 = 5,
        tpm                 = 20_000,
        daily_token_limit   = 50_000,
        monthly_token_limit = 500_000,
        monthly_budget_usd  = 2.0,
        max_context_length  = 4_096,
        concurrent_requests = 1,
        models_allowed      = ["llama3.1-8b", "mistral-7b"],
    ),
    "pro": PlanLimits(
        plan                = "pro",
        rpm                 = 60,
        tpm                 = 200_000,
        daily_token_limit   = 1_000_000,
        monthly_token_limit = 10_000_000,
        monthly_budget_usd  = 20.0,
        max_context_length  = 32_768,
        concurrent_requests = 5,
        models_allowed      = ["llama3.1-8b", "mistral-7b", "gpt-4o-mini"],
    ),
    "enterprise": PlanLimits(
        plan                = "enterprise",
        rpm                 = 600,
        tpm                 = 2_000_000,
        daily_token_limit   = 50_000_000,
        monthly_token_limit = 500_000_000,
        monthly_budget_usd  = 500.0,
        max_context_length  = 128_000,
        concurrent_requests = 50,
        models_allowed      = ["*"],   # all models
    ),
}

def get_plan_limits(plan: str) -> PlanLimits:
    return PLAN_LIMITS.get(plan, PLAN_LIMITS["free"])
```

---

## Phase 2: Redis Sliding Window Rate Limiter

### 2.1 Core Rate Limiter

Create `utils/rate_limiter.py`:

```python
import time
import redis.asyncio as aioredis
from fastapi import HTTPException

redis_client: aioredis.Redis = None  # initialized on startup

class RateLimitExceeded(HTTPException):
    def __init__(self, limit: int, window: int, retry_after: int):
        super().__init__(
            status_code=429,
            detail={
                "error":       "rate_limit_exceeded",
                "limit":       limit,
                "window_secs": window,
                "retry_after": retry_after,
            },
            headers={"Retry-After": str(retry_after)},
        )

async def check_rate_limit(
    key: str,
    limit: int,
    window_seconds: int = 60,
) -> tuple[int, int]:
    """
    Sliding window rate limiter using Redis sorted sets.
    Returns (current_count, remaining).
    Raises RateLimitExceeded if over limit.
    """
    now = time.time()
    window_start = now - window_seconds

    pipe = redis_client.pipeline()
    # Remove expired entries
    pipe.zremrangebyscore(key, 0, window_start)
    # Add current request with timestamp as score
    pipe.zadd(key, {f"{now}:{id(pipe)}": now})
    # Count requests in window
    pipe.zcard(key)
    # Set TTL so key auto-expires
    pipe.expire(key, window_seconds + 1)
    results = await pipe.execute()

    current_count = results[2]

    if current_count > limit:
        # Calculate when oldest request expires
        oldest = await redis_client.zrange(key, 0, 0, withscores=True)
        retry_after = int(window_seconds - (now - oldest[0][1])) + 1 if oldest else window_seconds
        raise RateLimitExceeded(limit=limit, window=window_seconds, retry_after=retry_after)

    remaining = max(0, limit - current_count)
    return current_count, remaining
```

### 2.2 Rate Limit Middleware

Create `middleware/rate_limit.py`:

```python
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import JSONResponse
from open_webui.utils.rate_limiter import check_rate_limit, RateLimitExceeded
from open_webui.config.plans import get_plan_limits

RATE_LIMITED_PATHS = {
    "/api/chat/completions",
    "/api/ollama/api/chat",
    "/api/openai/chat/completions",
}

class RateLimitMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        path = request.url.path

        if path not in RATE_LIMITED_PATHS:
            return await call_next(request)

        user = getattr(request.state, "user", None)
        if not user:
            return await call_next(request)

        plan = getattr(user, "plan", "free")
        limits = get_plan_limits(plan)

        try:
            # Per-user RPM check
            count, remaining = await check_rate_limit(
                key=f"rpm:{user.id}",
                limit=limits.rpm,
                window_seconds=60,
            )
        except RateLimitExceeded as e:
            return JSONResponse(
                status_code=429,
                content=e.detail,
                headers={"Retry-After": e.headers["Retry-After"]},
            )

        # Inject rate limit headers into response
        response = await call_next(request)
        response.headers["X-RateLimit-Limit"]     = str(limits.rpm)
        response.headers["X-RateLimit-Remaining"] = str(remaining)
        response.headers["X-RateLimit-Reset"]     = str(int(time.time()) + 60)
        return response
```

Register in `main.py`:
```python
app.add_middleware(RateLimitMiddleware)
```

---

## Phase 3: Concurrent Request Limiting

Prevent a single user from holding multiple simultaneous GPU slots:

### 3.1 Concurrent Request Tracker

Add to `utils/rate_limiter.py`:
```python
async def acquire_concurrent_slot(user_id: str, max_concurrent: int) -> bool:
    """
    Increment concurrent counter. Returns False if limit exceeded.
    Uses Redis atomic increment with TTL as safety net.
    """
    key = f"concurrent:{user_id}"
    current = await redis_client.incr(key)
    await redis_client.expire(key, 300)   # 5-min TTL as safety net

    if current > max_concurrent:
        await redis_client.decr(key)
        return False
    return True

async def release_concurrent_slot(user_id: str):
    key = f"concurrent:{user_id}"
    current = await redis_client.get(key)
    if current and int(current) > 0:
        await redis_client.decr(key)
```

### 3.2 Context Manager for Concurrent Slots

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def concurrent_slot(user_id: str, max_concurrent: int):
    acquired = await acquire_concurrent_slot(user_id, max_concurrent)
    if not acquired:
        raise HTTPException(
            status_code=429,
            detail={
                "error":   "concurrent_limit_exceeded",
                "message": f"You already have {max_concurrent} active request(s). Please wait for one to complete.",
            }
        )
    try:
        yield
    finally:
        await release_concurrent_slot(user_id)
```

Usage in chat endpoint:
```python
@router.post("/api/chat/completions")
async def chat_completion(request: ChatRequest, ctx = Depends(get_current_user_with_org)):
    limits = get_plan_limits(ctx.user.plan)
    async with concurrent_slot(ctx.user.id, limits.concurrent_requests):
        return await forward_to_litellm(request, ctx)
```

---

## Phase 4: Token Usage Tracking

### 4.1 Usage Log Table

```sql
CREATE TABLE usage_logs (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID REFERENCES users(id),
  organization_id UUID REFERENCES organizations(id),
  model           VARCHAR(128),
  prompt_tokens   INTEGER DEFAULT 0,
  completion_tokens INTEGER DEFAULT 0,
  total_tokens    INTEGER DEFAULT 0,
  request_duration_ms INTEGER,
  status          VARCHAR(32) DEFAULT 'success',  -- success | error | rate_limited
  created_at      TIMESTAMP DEFAULT NOW()
);

-- Optimized for quota queries
CREATE INDEX idx_usage_user_day   ON usage_logs(user_id, DATE(created_at));
CREATE INDEX idx_usage_user_month ON usage_logs(user_id, DATE_TRUNC('month', created_at));
CREATE INDEX idx_usage_org_month  ON usage_logs(organization_id, DATE_TRUNC('month', created_at));
```

### 4.2 Usage Logger Service

Create `services/usage_tracker.py`:
```python
async def log_usage(
    user_id:          str,
    org_id:           str,
    model:            str,
    prompt_tokens:    int,
    completion_tokens: int,
    duration_ms:      int,
    status:           str = "success",
):
    total = prompt_tokens + completion_tokens
    await db.execute(
        """INSERT INTO usage_logs
           (user_id, organization_id, model, prompt_tokens, completion_tokens,
            total_tokens, request_duration_ms, status)
           VALUES ($1,$2,$3,$4,$5,$6,$7,$8)""",
        user_id, org_id, model, prompt_tokens, completion_tokens, total, duration_ms, status
    )
    # Also update Redis daily/monthly counters for fast quota checks
    await redis_client.incrby(f"tokens:day:{user_id}:{today()}", total)
    await redis_client.expire(f"tokens:day:{user_id}:{today()}", 86400 * 2)
    await redis_client.incrby(f"tokens:month:{user_id}:{month()}", total)
    await redis_client.expire(f"tokens:month:{user_id}:{month()}", 86400 * 32)

async def get_daily_tokens(user_id: str) -> int:
    val = await redis_client.get(f"tokens:day:{user_id}:{today()}")
    return int(val) if val else 0

async def get_monthly_tokens(user_id: str) -> int:
    val = await redis_client.get(f"tokens:month:{user_id}:{month()}")
    return int(val) if val else 0
```

### 4.3 Token Quota Check (Pre-Request)

Check quota before forwarding to LiteLLM:
```python
async def check_token_quota(user_id: str, plan: str):
    limits = get_plan_limits(plan)

    daily_used   = await get_daily_tokens(user_id)
    monthly_used = await get_monthly_tokens(user_id)

    if daily_used >= limits.daily_token_limit:
        raise HTTPException(429, {
            "error":    "daily_token_quota_exceeded",
            "used":     daily_used,
            "limit":    limits.daily_token_limit,
            "resets_in": seconds_until_midnight(),
        })

    if monthly_used >= limits.monthly_token_limit:
        raise HTTPException(429, {
            "error":    "monthly_token_quota_exceeded",
            "used":     monthly_used,
            "limit":    limits.monthly_token_limit,
            "resets_in": seconds_until_month_end(),
        })
```

---

## Phase 5: Nginx IP-Level Protection

### 5.1 Nginx Rate Limit Config

```nginx
# /etc/nginx/nginx.conf

# Define rate limit zones
limit_req_zone $binary_remote_addr zone=api_per_ip:10m rate=30r/m;
limit_req_zone $binary_remote_addr zone=auth_per_ip:10m rate=5r/m;
limit_conn_zone $binary_remote_addr zone=conn_per_ip:10m;

server {
    listen 443 ssl;
    server_name yoursaas.com;

    # Auth endpoints — strict
    location ~ ^/api/auth/ {
        limit_req  zone=auth_per_ip burst=3 nodelay;
        limit_conn conn_per_ip 5;
        proxy_pass http://openwebui;
    }

    # Chat API — moderate
    location ~ ^/api/chat/ {
        limit_req  zone=api_per_ip burst=10 nodelay;
        limit_conn conn_per_ip 10;
        proxy_pass http://openwebui;
    }

    # Static assets — unlimited
    location /static/ {
        proxy_pass http://openwebui;
    }

    # Return 429 instead of 503 for rate limited requests
    limit_req_status 429;
    limit_conn_status 429;
}
```

---

## Phase 6: LiteLLM-Level Rate Limiting

LiteLLM enforces per-virtual-key limits as the final layer. Set when creating virtual keys (see LLM Load Balancing plan):

```python
{
    "tpm_limit": get_tpm_for_plan(plan),    # tokens per minute
    "rpm_limit": get_rpm_for_plan(plan),    # requests per minute
    "max_budget": monthly_budget_usd,        # total USD spend cap
    "budget_duration": "1mo",               # reset monthly
}
```

LiteLLM returns `429` with a `Retry-After` header when limits are hit, which Open WebUI surfaces to the user.

---

## Phase 7: User-Facing Quota Dashboard

### 7.1 New API Endpoint

`GET /api/user/quota`:
```python
@router.get("/api/user/quota")
async def get_user_quota(ctx = Depends(get_current_user_with_org)):
    limits = get_plan_limits(ctx.user.plan)
    daily_used   = await get_daily_tokens(ctx.user.id)
    monthly_used = await get_monthly_tokens(ctx.user.id)

    return {
        "plan":  ctx.user.plan,
        "daily": {
            "used":      daily_used,
            "limit":     limits.daily_token_limit,
            "remaining": max(0, limits.daily_token_limit - daily_used),
            "resets_in": seconds_until_midnight(),
        },
        "monthly": {
            "used":      monthly_used,
            "limit":     limits.monthly_token_limit,
            "remaining": max(0, limits.monthly_token_limit - monthly_used),
            "resets_in": seconds_until_month_end(),
        },
        "rate_limits": {
            "rpm":               limits.rpm,
            "concurrent":        limits.concurrent_requests,
            "max_context_length": limits.max_context_length,
        },
    }
```

### 7.2 Frontend Quota Widget

Add a quota indicator to the Open WebUI sidebar:
```svelte
<!-- src/lib/components/QuotaIndicator.svelte -->
<script>
  import { onMount } from 'svelte';
  let quota = null;

  onMount(async () => {
    const res = await fetch('/api/user/quota');
    quota = await res.json();
  });
</script>

{#if quota}
  <div class="quota-bar">
    <span>Daily: {formatTokens(quota.daily.used)} / {formatTokens(quota.daily.limit)}</span>
    <progress value={quota.daily.used} max={quota.daily.limit}></progress>
    {#if quota.daily.remaining < quota.daily.limit * 0.1}
      <span class="warning">⚠ Almost at daily limit</span>
    {/if}
  </div>
{/if}
```

---

## Phase 8: Abuse Detection

### 8.1 Anomaly Detection Rules

Create `services/abuse_detector.py`:
```python
ABUSE_RULES = [
    # Rapid-fire requests despite rate limiting
    {
        "name":        "repeated_rate_limit_hits",
        "condition":   lambda stats: stats["rate_limit_hits_1h"] > 50,
        "action":      "warn_user",
    },
    # Extremely long prompts (likely trying to overflow)
    {
        "name":        "prompt_flooding",
        "condition":   lambda req: req.total_prompt_tokens > 100_000,
        "action":      "block_request",
    },
    # Account used from many IPs in short time
    {
        "name":        "ip_rotation",
        "condition":   lambda stats: stats["unique_ips_1h"] > 10,
        "action":      "flag_for_review",
    },
]

async def evaluate_request(user_id: str, request_meta: dict):
    stats = await get_user_abuse_stats(user_id)
    for rule in ABUSE_RULES:
        if rule["condition"](stats):
            await handle_abuse(user_id, rule["name"], rule["action"])
```

### 8.2 Actions

- `warn_user` — email + in-app notice
- `block_request` — reject with `403`
- `flag_for_review` — add to ops queue, auto-suspend after 3 flags
- `suspend_account` — immediate account lock, requires manual review

---

## Environment Variables Reference

```bash
# Redis for rate limiting
REDIS_URL=redis://redis:6379/0

# Rate limit tuning overrides (optional — defaults from plans.py)
FREE_PLAN_RPM=5
PRO_PLAN_RPM=60
ENTERPRISE_PLAN_RPM=600

# Abuse thresholds
ABUSE_RATE_LIMIT_HITS_THRESHOLD=50
ABUSE_UNIQUE_IP_THRESHOLD=10
```

---

## Implementation Checklist

- [ ] `plans.py` defines all tier limits
- [ ] Redis sliding window RPM check working (test with `hey` or `k6`)
- [ ] Concurrent request limiter tested: free user blocked at 2nd simultaneous request
- [ ] Token tracking writing to `usage_logs` table
- [ ] Daily + monthly Redis counters incrementing correctly
- [ ] `/api/user/quota` endpoint returns correct data
- [ ] Nginx `limit_req` zones deployed and returning 429
- [ ] LiteLLM virtual keys have correct `tpm_limit` and `max_budget`
- [ ] Frontend quota bar renders and shows warning near limit
- [ ] Abuse detection flags account after 50 rate-limit hits in 1 hour
- [ ] Rate limit 429 response includes `Retry-After` header
- [ ] All limit values configurable via environment variables
