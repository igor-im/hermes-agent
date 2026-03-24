# Hermes Agent — Architecture, Security & Railway Deployment Report

**Date:** 2026-03-24

---

## 1. Architecture Overview

### Core Design Pattern

Hermes is a **multi-platform AI agent gateway** — it normalizes messages from 15+ chat platforms into a unified `MessageEvent`, routes them through a Claude-powered AI agent with a dynamic tool registry, and delivers responses back across platforms.

```
Platforms (Telegram, Discord, Slack, …)
    ↓ MessageEvent normalization
GatewayRunner (lifecycle, agent caching, memory)
    ↓
AIAgent (Claude API, context compression, model routing)
    ↓
ToolRegistry (22 built-in tool modules, MCP servers)
    ↓
DeliveryRouter (multi-destination, deduplication)
    ↓
Platform adapters (outbound)
```

### Key Architectural Patterns

| Pattern | Implementation | Notes |
|---|---|---|
| **Tool Registry** | Singleton `ToolRegistry` with `ToolEntry` dataclass, dynamic module discovery | Clean separation; `check_fn()` callbacks for conditional availability |
| **Async Bridging** | Three-path model: async context, worker thread, main thread via persistent per-thread event loops | Handles sync→async correctly but adds complexity |
| **Agent Caching** | `_agent_cache[session_key] → (AIAgent, config_signature)` | Preserves Claude prompt caching across turns; invalidates on config change |
| **Session Management** | PII-aware hashing (SHA256 → 12-char hex) with platform-specific strategies | Full redaction for WhatsApp/Signal/Telegram; preserved IDs for Discord mentions |
| **Message Delivery** | `DeliveryRouter` with target parsing ("origin", "local", "telegram:123456") | Deduplication, per-target error handling, 4000-char truncation |
| **Context Compression** | Automatic at 85% context usage via auxiliary LLM call | Prevents context overflow gracefully |
| **Terminal Isolation** | 6 backends: local, Docker, SSH, Daytona, Singularity, Modal | Excellent sandboxing options for Railway |
| **Hooks** | Filesystem-based discovery (`~/.hermes/hooks/`), wildcard event matching | Error-resilient — exceptions logged, never blocking |

### Modularity Assessment

The architecture is well-modularized:

- Clear separation between gateway lifecycle, platform adapters, tool system, and delivery
- Platform adapters follow a consistent base class pattern
- Tool registry is decoupled from tool implementations
- Config-driven behavior throughout

**Weakness**: `GatewayRunner` (`gateway/run.py`, 650+ lines) handles lifecycle, caching, memory flushing, and adapter management — a candidate for decomposition.

---

## 2. Security Findings

### CRITICAL

#### 1. Timing-Unsafe Token Comparison

- **File:** `gateway/platforms/api_server.py:279`
- **Issue:** `token == self._api_key` — vulnerable to timing side-channel attack
- **Risk:** Attacker can extract the API key byte-by-byte via response time analysis
- **Fix:** Replace with `hmac.compare_digest(token, self._api_key)`
- **Railway relevance:** HIGH — Railway exposes HTTP endpoints publicly; this is directly exploitable

#### 2. Sensitive Data in Error Responses

- **File:** `gateway/platforms/api_server.py:663`
- **Risk:** Internal state/config details leak in error payloads
- **Fix:** Return generic error messages; log details server-side only

### HIGH

#### 3. DNS Rebinding TOCTOU in SSRF Protection

- **File:** `tools/url_safety.py`
- **Issue:** URL safety checks resolve DNS then allow the request — attacker can rebind DNS between check and fetch
- **Mitigation:** Pin resolved IP and pass to the HTTP client, or use a connect-level hook

#### 4. `.env.example` Unusually Large (14.3K)

- Verify no real credentials have been committed. Could not read due to permissions.

### MEDIUM

#### 5. No Rate Limiting on API Server

- Railway endpoints are public; without rate limiting, Claude API billing is exposed to abuse.

#### 6. Hook Handlers Execute Arbitrary Code

- Filesystem-based discovery means anyone with write access to `~/.hermes/hooks/` can inject code. On Railway this is low risk (container isolation), but relevant if volumes are mounted.

### Positive Security Patterns

- Five-layer defense model (authorization → approval → container → MCP filtering → context scanning)
- SSRF protection with private IP/CGNAT/metadata blocking and fail-closed DNS
- OAuth 2.1 PKCE with `chmod 0600` token storage
- PII-aware session hashing
- Path traversal protection via `.is_relative_to()` + UUID prefixing on file caches
- Docker hardening flags documented in security guide

---

## 3. Railway-Specific Deployment Considerations

### Port Binding

The API server reads from `API_SERVER_PORT` env var (default `8642`). **Railway injects `PORT`** — you must either:

- Set `API_SERVER_PORT=$PORT` in Railway environment variables, or
- Update `gateway/platforms/api_server.py:209` to also check `PORT`

### Health Check Endpoint

No health check endpoint exists in the current API server. The planned OpenAI-compatible server (`.plans/openai-api-server.md`) includes `GET /health`, but it is not implemented. **Railway needs a health check endpoint** for deployment monitoring and zero-downtime deploys.

### Container Configuration

No `Dockerfile`, `Procfile`, or `nixpacks.toml` exists. Railway will attempt auto-detection via Nixpacks (Python/pip), but explicit configuration is recommended for:

- Build dependencies (especially for optional extras like `browser-tools`, `vision`)
- Startup command
- Python version pinning (3.11+ required)

### Persistent Storage

Railway containers are **ephemeral** — no persistent filesystem. This affects:

| Component | Impact | Mitigation |
|---|---|---|
| Hook discovery (`~/.hermes/hooks/`) | Hooks lost on redeploy | Bake into container image |
| OAuth token storage (`HermesTokenStorage`) | Tokens lost on redeploy | External store (Redis/Postgres) |
| Skill storage (autonomous creation) | Skills lost on redeploy | Disable auto-creation or persist externally |
| Session/memory state | Lost if file-based | Use Honcho or external DB |
| Config file (`cli-config.yaml`) | Must be reproduced | Set via env vars or bake into image |

### Networking

- Railway provides HTTPS termination — no need for TLS in the app
- Outbound connections (Claude API, platform webhooks) work without restrictions
- Private networking available if running multiple Railway services

### Terminal Isolation

Docker-in-Docker won't work on Railway without special configuration. Consider:

- `modal` backend — remote sandboxing, no Docker dependency
- `daytona` backend — remote workspace isolation
- `local` — acceptable if the agent's tool use is trusted/gated

---

## 4. Actionable Recommendations

### Must Fix Before Deploy (Critical)

| # | Issue | File | Fix |
|---|---|---|---|
| 1 | Timing-unsafe token comparison | `api_server.py:279` | `hmac.compare_digest()` |
| 2 | Sensitive data in error responses | `api_server.py:663` | Generic error messages; log internally |
| 3 | No health check endpoint | `api_server.py` | Add `GET /health` returning 200 |
| 4 | `PORT` env var not supported | `api_server.py:209` | Fall back to `PORT` if `API_SERVER_PORT` unset |

### Should Fix (High Priority)

| # | Issue | Recommendation |
|---|---|---|
| 5 | No rate limiting | Add rate limiting middleware (per-IP, per-token) to protect Claude API billing |
| 6 | DNS rebinding TOCTOU | Pin resolved IPs in `url_safety.py` |
| 7 | No Dockerfile | Create a production Dockerfile with multi-stage build |
| 8 | No `.aisteering/` directory | Policy violation per global CLAUDE.md; should be created |

### Railway Setup Checklist

- [ ] Create `Dockerfile` or `railway.toml` with explicit build config
- [ ] Map `PORT` → `API_SERVER_PORT` in Railway environment variables
- [ ] Add health check endpoint and configure Railway's health check path
- [ ] Move all config to environment variables (Railway injects these)
- [ ] Set up Railway Postgres or Redis for persistent state (sessions, OAuth tokens)
- [ ] Configure Railway's restart policy for crash recovery
- [ ] Use Railway's private networking if running multiple services
- [ ] Set `HERMES_LOG_LEVEL` and ensure no sensitive data in stdout (Railway captures logs)
- [ ] Choose appropriate terminal isolation backend (`modal` or `daytona` recommended over Docker)
- [ ] Review cold start time — 22 tool modules import at discovery; consider lazy loading

---

## 5. Architecture Strengths & Risks

### Strengths

- Well-separated concerns with clean module boundaries
- Platform adapter pattern scales well for new integrations
- Agent caching with config-signature invalidation is efficient
- Five-layer security model is thorough
- Context compression prevents runaway costs
- Smart model routing with fallback provides resilience
- Six terminal isolation backends offer deployment flexibility

### Risks

| Risk | Severity | Notes |
|---|---|---|
| `GatewayRunner` overloaded | Medium | 650+ lines, multiple responsibilities — decomposition candidate |
| Skills auto-creation at 15 iterations | Medium | Self-modification loop — disable or gate in production |
| Memory flushing via pre-reset agent turn | Low | Adds latency and cost to every session reset |
| 22 tool modules import at discovery | Low | Cold start penalty on ephemeral containers |
| No structured logging | Medium | Makes Railway log analysis harder; consider JSON logging |
