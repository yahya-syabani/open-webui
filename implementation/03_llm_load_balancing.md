# Implementation Plan: LLM Load Balancing
**Antigravity IDE — Open WebUI SaaS Layer**
**Document Version:** 1.0
**Focus Area:** Multi-model routing, GPU load distribution, failover, LiteLLM integration

---

## Overview

A single Ollama instance or LLM endpoint is the primary bottleneck in any AI SaaS. When multiple users send concurrent chat requests, they queue up behind a single model process, creating unacceptable latency. This plan covers integrating LiteLLM as a smart load-balancing proxy between Open WebUI and all LLM backends, enabling horizontal scaling of inference, automatic failover, and model-tier routing.

---

## Current State (Open WebUI Baseline)

- Open WebUI sends chat completions to a single `OLLAMA_BASE_URL`
- No fallback if Ollama is unreachable
- No concept of routing by model tier, user plan, or load
- OpenAI API also supported but configured as a single endpoint
- No per-user or per-model request queuing

---

## Target Architecture

```
Open WebUI Instances
        │
        ▼
  LiteLLM Proxy (port 4000)
        │
   ┌────┴──────────────────────────────────┐
   │           Router Logic                 │
   │  - Round robin                         │
   │  - Least-busy                          │
   │  - Cost-optimized fallback             │
   │  - Model-tier routing                  │
   └────┬──────────┬──────────┬────────────┘
        │          │          │
   Ollama #1   Ollama #2   OpenAI API
  (GPU node1) (GPU node2) (overflow/fallback)
        │          │
   [llama3.1]  [mistral]  [gpt-4o, claude-sonnet]
```

---

## Phase 1: LiteLLM Proxy Setup

### 1.1 Installation

```bash
pip install litellm[proxy]
# or via Docker (recommended for production)
docker pull ghcr.io/berriai/litellm:main-latest
```

### 1.2 Core Configuration (`litellm_config.yaml`)

```yaml
model_list:
  # Ollama Node 1 — primary GPU
  - model_name: llama3.1-8b
    litellm_params:
      model: ollama/llama3.1:8b
      api_base: http://ollama-node1:11434
      timeout: 120
      stream_timeout: 300

  - model_name: llama3.1-8b
    litellm_params:
      model: ollama/llama3.1:8b
      api_base: http://ollama-node2:11434
      timeout: 120
      stream_timeout: 300

  # Ollama Node 2 — secondary GPU
  - model_name: mistral-7b
    litellm_params:
      model: ollama/mistral:7b
      api_base: http://ollama-node1:11434

  - model_name: mistral-7b
    litellm_params:
      model: ollama/mistral:7b
      api_base: http://ollama-node2:11434

  # OpenAI fallback / premium tier
  - model_name: gpt-4o
    litellm_params:
      model: openai/gpt-4o
      api_key: os.environ/OPENAI_API_KEY

  - model_name: claude-sonnet
    litellm_params:
      model: anthropic/claude-sonnet-4-6
      api_key: os.environ/ANTHROPIC_API_KEY

router_settings:
  routing_strategy: least-busy       # options: simple-shuffle, least-busy, latency-based, cost-based
  num_retries: 3
  retry_after: 5
  allowed_fails: 2                   # mark model unhealthy after 2 consecutive failures
  cooldown_time: 30                  # seconds before retrying unhealthy model
  timeout: 120
  stream_timeout: 300

litellm_settings:
  drop_params: true                  # ignore unsupported params per model
  set_verbose: false
  json_logs: true
  callbacks: ["prometheus"]         # metrics export

general_settings:
  master_key: os.environ/LITELLM_MASTER_KEY
  database_url: os.environ/DATABASE_URL   # LiteLLM uses same PG for spend tracking
  store_model_in_db: true
```

### 1.3 Run LiteLLM Proxy

```bash
litellm --config litellm_config.yaml --port 4000 --detailed_debug
```

Or via Docker:
```yaml
# docker-compose.yml
litellm:
  image: ghcr.io/berriai/litellm:main-latest
  ports:
    - "4000:4000"
  environment:
    LITELLM_MASTER_KEY: ${LITELLM_MASTER_KEY}
    OPENAI_API_KEY: ${OPENAI_API_KEY}
    ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
    DATABASE_URL: ${DATABASE_URL}
  volumes:
    - ./litellm_config.yaml:/app/config.yaml
  command: ["--config", "/app/config.yaml", "--port", "4000"]
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:4000/health"]
    interval: 30s
    timeout: 10s
    retries: 3
```

---

## Phase 2: Open WebUI → LiteLLM Integration

### 2.1 Update Open WebUI Config

Change from pointing to Ollama directly to pointing at LiteLLM:
```bash
# Before
OLLAMA_BASE_URL=http://ollama:11434

# After — LiteLLM exposes an OpenAI-compatible API
OPENAI_API_BASE_URL=http://litellm:4000
OPENAI_API_KEY=${LITELLM_MASTER_KEY}
```

### 2.2 Per-User Virtual Keys

LiteLLM supports virtual API keys with spend limits and model access rules. Generate one per user/org during onboarding:

```python
# backend/open_webui/services/litellm.py

import httpx

LITELLM_BASE_URL = os.environ["LITELLM_BASE_URL"]
LITELLM_MASTER_KEY = os.environ["LITELLM_MASTER_KEY"]

async def create_virtual_key(
    user_id: str,
    org_id: str,
    plan: str,
    monthly_budget_usd: float = 10.0
) -> str:
    """Create a LiteLLM virtual key scoped to this user's plan."""
    model_access = {
        "free":       ["llama3.1-8b", "mistral-7b"],
        "pro":        ["llama3.1-8b", "mistral-7b", "gpt-4o"],
        "enterprise": ["*"],
    }.get(plan, ["llama3.1-8b"])

    async with httpx.AsyncClient() as client:
        resp = await client.post(
            f"{LITELLM_BASE_URL}/key/generate",
            headers={"Authorization": f"Bearer {LITELLM_MASTER_KEY}"},
            json={
                "key_alias":     f"user:{user_id}",
                "team_id":       org_id,
                "models":        model_access,
                "max_budget":    monthly_budget_usd,
                "budget_duration": "1mo",
                "metadata":      {"user_id": user_id, "plan": plan},
                "tpm_limit":     get_tpm_for_plan(plan),
                "rpm_limit":     get_rpm_for_plan(plan),
            }
        )
        return resp.json()["key"]

def get_tpm_for_plan(plan: str) -> int:
    return {"free": 50_000, "pro": 500_000, "enterprise": 5_000_000}.get(plan, 50_000)

def get_rpm_for_plan(plan: str) -> int:
    return {"free": 10, "pro": 100, "enterprise": 1000}.get(plan, 10)
```

Store the virtual key in the user's record. Open WebUI uses this key when making requests on behalf of that user.

---

## Phase 3: Routing Strategies

### 3.1 Routing Strategy Selection

| Strategy | Config Value | Best For |
|---|---|---|
| Round-robin across replicas | `simple-shuffle` | Even load, same-spec nodes |
| Least active requests | `least-busy` | Variable request duration (streaming) |
| Lowest latency | `latency-based-routing` | Latency-sensitive users |
| Cost-minimizing | `cost-based-routing` | Mixed cloud + local deployment |

For most SaaS setups: **`least-busy`** is recommended because LLM streaming requests vary heavily in duration.

### 3.2 Model Tier Routing

Route different user plans to different model tiers automatically:

```yaml
# In litellm_config.yaml — define model groups
model_list:
  - model_name: "free-tier"
    litellm_params:
      model: ollama/llama3.1:8b
      api_base: http://ollama-node1:11434

  - model_name: "pro-tier"
    litellm_params:
      model: openai/gpt-4o-mini
      api_key: os.environ/OPENAI_API_KEY

  - model_name: "enterprise-tier"
    litellm_params:
      model: openai/gpt-4o
      api_key: os.environ/OPENAI_API_KEY
```

In Open WebUI, model selection can be constrained per user role/plan via the virtual key `models` list.

### 3.3 Fallback Chain

Define fallback order — if primary fails, cascade down:
```yaml
litellm_settings:
  fallbacks:
    - {"llama3.1-8b": ["mistral-7b", "gpt-4o-mini"]}
    - {"gpt-4o": ["claude-sonnet", "gpt-4o-mini"]}

  context_window_fallbacks:
    - {"llama3.1-8b": ["mistral-7b"]}
```

This means: if `llama3.1-8b` fails, try `mistral-7b`, then fall back to `gpt-4o-mini`.

---

## Phase 4: Ollama Multi-Node Setup

### 4.1 Multiple Ollama Instances

Each GPU node runs an independent Ollama instance:

```yaml
# docker-compose.yml (GPU nodes)
ollama-node1:
  image: ollama/ollama:latest
  runtime: nvidia
  environment:
    NVIDIA_VISIBLE_DEVICES: "0"
    OLLAMA_NUM_PARALLEL: 4        # parallel requests per model
    OLLAMA_MAX_LOADED_MODELS: 2   # models kept in VRAM
    OLLAMA_FLASH_ATTENTION: 1
  volumes:
    - ollama1_data:/root/.ollama
  ports:
    - "11434:11434"

ollama-node2:
  image: ollama/ollama:latest
  runtime: nvidia
  environment:
    NVIDIA_VISIBLE_DEVICES: "1"
    OLLAMA_NUM_PARALLEL: 4
    OLLAMA_MAX_LOADED_MODELS: 2
    OLLAMA_FLASH_ATTENTION: 1
  volumes:
    - ollama2_data:/root/.ollama
  ports:
    - "11435:11434"
```

### 4.2 Ollama Performance Tuning

Key environment variables for throughput:
```bash
OLLAMA_NUM_PARALLEL=4        # 4 simultaneous completions per model
OLLAMA_MAX_QUEUE=512         # internal request queue depth
OLLAMA_KEEP_ALIVE=5m         # keep model in VRAM for 5 minutes
OLLAMA_FLASH_ATTENTION=1     # enable flash attention (reduces VRAM, faster)
OLLAMA_KV_CACHE_TYPE=q8_0    # quantize KV cache (more users per GPU)
```

### 4.3 Model Pre-loading

Pre-pull models on startup so first request isn't slow:
```bash
#!/bin/bash
# startup-pull.sh — run once per Ollama node
ollama pull llama3.1:8b
ollama pull mistral:7b
ollama pull nomic-embed-text   # for RAG embeddings
```

---

## Phase 5: Queue Management for Burst Traffic

### 5.1 Request Queue with Redis

For burst scenarios where GPU is fully saturated, queue requests instead of returning errors:

Install: `arq` (async Redis queue)

Create `workers/llm_worker.py`:
```python
from arq import create_pool
from arq.connections import RedisSettings

async def process_chat_request(ctx, user_id: str, messages: list, model: str):
    """Worker function — dequeued and executed by worker process."""
    result = await call_litellm(model=model, messages=messages)
    # Store result in Redis for polling or push via WebSocket
    await ctx["redis"].setex(f"chat_result:{user_id}:{job_id}", 300, json.dumps(result))
    return result

class WorkerSettings:
    functions = [process_chat_request]
    redis_settings = RedisSettings.from_dsn(os.environ["REDIS_URL"])
    max_jobs = 10          # concurrent jobs per worker process
    job_timeout = 300      # 5 min max per job
    queue_read_limit = 100
```

### 5.2 Queue Depth Monitoring

Alert when queue depth exceeds threshold:
```python
async def get_queue_depth() -> int:
    redis = await create_pool(RedisSettings.from_dsn(REDIS_URL))
    return await redis.zcard("arq:default")

# Alert if > 50 queued requests
if await get_queue_depth() > 50:
    send_alert("LLM queue depth critical — consider scaling up GPU nodes")
```

---

## Phase 6: Health Checks & Circuit Breaker

### 6.1 LiteLLM Health Endpoint

LiteLLM exposes `/health` and `/health/liveliness`:
```bash
curl http://litellm:4000/health
# Returns per-model health status
```

### 6.2 Custom Health Check Service

Create `services/health_monitor.py` — periodically checks each backend:
```python
async def check_ollama_health(base_url: str) -> bool:
    try:
        async with httpx.AsyncClient(timeout=5) as client:
            resp = await client.get(f"{base_url}/api/tags")
            return resp.status_code == 200
    except Exception:
        return False

# Run every 30 seconds
@scheduler.scheduled_job("interval", seconds=30)
async def monitor_backends():
    for node_url in OLLAMA_NODES:
        healthy = await check_ollama_health(node_url)
        await redis.setex(f"ollama:health:{node_url}", 60, "1" if healthy else "0")
        if not healthy:
            await alert_ops(f"Ollama node {node_url} is unhealthy")
```

### 6.3 Circuit Breaker

LiteLLM's `allowed_fails` and `cooldown_time` implement a basic circuit breaker. For custom logic:
```python
async def is_backend_healthy(model_name: str) -> bool:
    key = f"circuit:{model_name}"
    state = await redis.get(key)
    return state != "open"

async def trip_circuit(model_name: str, cooldown_seconds: int = 60):
    await redis.setex(f"circuit:{model_name}", cooldown_seconds, "open")
```

---

## Phase 7: Observability

### 7.1 LiteLLM Prometheus Metrics

LiteLLM exports Prometheus metrics at `/metrics`:
- `litellm_requests_total` — total requests per model
- `litellm_request_duration_seconds` — latency histogram
- `litellm_tokens_used_total` — tokens per model per user
- `litellm_spend_total` — cost tracking

### 7.2 Grafana Dashboard Panels

Key panels to build:
- Requests/sec per model (realtime)
- p95 latency per model
- GPU queue depth per Ollama node
- Token throughput (tokens/sec)
- Monthly spend by model, org, user
- Error rate and circuit breaker status

### 7.3 Alerting Rules

```yaml
# prometheus-alerts.yml
groups:
  - name: llm-load-balancer
    rules:
      - alert: HighLLMLatency
        expr: histogram_quantile(0.95, litellm_request_duration_seconds) > 30
        for: 5m
        annotations:
          summary: "LLM p95 latency > 30s"

      - alert: LLMErrorRateHigh
        expr: rate(litellm_requests_total{status="error"}[5m]) > 0.1
        for: 2m
        annotations:
          summary: "LLM error rate > 10%"

      - alert: OllamaNodeDown
        expr: up{job="ollama"} == 0
        for: 1m
        annotations:
          summary: "Ollama node is down"
```

---

## Environment Variables Reference

```bash
# LiteLLM
LITELLM_BASE_URL=http://litellm:4000
LITELLM_MASTER_KEY=sk-your-master-key-here

# Model backend endpoints
OLLAMA_NODE1_URL=http://ollama-node1:11434
OLLAMA_NODE2_URL=http://ollama-node2:11434

# Cloud fallbacks
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
TOGETHER_API_KEY=...

# LiteLLM DB (for spend tracking — can share with app DB)
LITELLM_DATABASE_URL=postgresql://owui:password@pgbouncer:5432/openwebui
```

---

## Implementation Checklist

- [ ] LiteLLM proxy running and reachable from Open WebUI
- [ ] Open WebUI `OPENAI_API_BASE_URL` pointing to LiteLLM
- [ ] All Ollama nodes registered in `litellm_config.yaml`
- [ ] `least-busy` routing confirmed working (send 10 concurrent requests, check distribution)
- [ ] Fallback chain tested: kill primary Ollama, verify fallback activates
- [ ] Virtual keys created per user on signup
- [ ] Per-plan model access restrictions enforced
- [ ] Prometheus metrics flowing to Grafana
- [ ] Health check alerting configured
- [ ] Circuit breaker tested: node down → auto-remove → cooldown → re-add
- [ ] GPU queue depth alert set at 50 pending requests
