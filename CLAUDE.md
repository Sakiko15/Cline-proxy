# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

`cline-go-proxy` is a single-binary Go reverse proxy that exposes one HTTP server to Cline clients while fronting two upstreams, routed purely by requested model name:

- **Cline account pool** — billed models via `api.cline.bot`, with multi-account rotation, OAuth login, cooldowns, and load balancing
- **Zen gateway** — free opencode models via `opencode.ai/zen/v1` (anonymous key), with proxy-pool egress, rate-limit defense, and context compaction

Clients can speak OpenAI Chat Completions, Anthropic Messages, or OpenAI Responses API. A Chinese admin panel (embedded SPA, no auth) lives at `/admin/`.

## Commands

```bash
go run .                    # start proxy, listens 0.0.0.0:3457
go build -o cline-proxy.exe .
./cline-proxy.exe -host 127.0.0.1 -port 3457
go run . -start             # build + start + open admin panel in browser
go run . -login             # OAuth device login, adds account to pool
go run . -list              # list accounts in pool
docker compose up -d        # containerized; data/ + override.md bind-mounted
```

- **Tests**: none exist — `go test ./...` runs zero tests. No lint tooling, no Makefile.
- All CLI flags are defined in `main.go`. The Dockerfile's `ENV PORT` is decorative — the `CMD -port` argument wins.
- Pushing to `main` builds 6 platform binaries and auto-creates a GitHub Release with the next `v*` patch version.

## Architecture

Entry: `main.go` → `app.StartProxy` (`internal/app/proxy.go:48`), which pre-warms account tokens, starts background goroutines (Cline model sync, zen model sync, compact-state cleanup), and serves a single `ServeMux` wrapped in request-log middleware. No web framework.

**All server logic is one package, `internal/app`** (~7.7k LOC). `internal/cline` is the WorkOS device-OAuth client; `internal/kit` holds data-path resolution and shared HTTP/identity helpers.

### Routing: model name is the only dispatch key

Every API handler is wrapped in `apiKeyHandler` (proxy.go:107): if the pool has any configured API keys they are enforced (`x-api-key` or `Authorization: Bearer`); no keys → open access.

`routeModel` (`internal/app/zen.go:108`) decides the upstream:
- free zen models (seed whitelist or `-free` suffix) → zen gateway
- paid zen models → explicit 400 rejection
- everything else → Cline pool

When zen is degraded (3 consecutive failures within 5 minutes), free-model requests fail over to the Cline pool.

### Cline path

`callClineAPI` (proxy.go:515): pick account → ensure fresh token → build OpenAI body (`buildUpstreamBody`, proxy.go:452 — `max_tokens` 128000, `reasoning_effort` "high", generated `session_id`) → POST to `api.cline.bot` with spoofed Cline 3.0.50 client headers. 401 → refresh token and retry once; 429 → parse `"Try again in 17h 59m"` (`parseInferenceCapDuration`, proxy.go:1953) or `Retry-After` → account cooldown.

### Protocol translation: all wire formats converge on OpenAI `tool_calls`

- Anthropic → OpenAI: `anthropicToOpenAI` (proxy.go:993), including `tool_use`/`tool_result` blocks → `tool_calls`/`role:tool`
- OpenAI → Anthropic: `openAIToAnthropic` (proxy.go:1246)
- Responses API: `responsesToChat` / `chatToResponses` / `chatStreamToResponses` (responses.go:20/156/264)

Streaming is line-by-line SSE in both directions. The Anthropic stream synthesizes a full `message_start`/`content_block_*`/`message_delta`/`message_stop` sequence, accumulates partial tool-call args, and reparses them with a tolerant parser (`parseToolArgs`, proxy.go:1106) before pruning args to the client's declared schemas (`filterToolInput`, proxy.go:1207). Models flagged `RequiresStream` force an upstream stream even for non-stream requests.

### Zen path (the most elaborate subsystem)

`internal/app/zen.go` + `proxy_pool.go`: concurrency semaphore, jittered exponential backoff honoring `Retry-After`, per-proxy cooldowns (round_robin/random/fill strategies), identity rotation (random `x-opencode-session`/UA per request, `internal/kit/ident.go`), and a uTLS Chrome-120 + HTTP/2 transport (`buildZenTransport`, proxy_pool.go:131) to pass Cloudflare fingerprinting. `compact.go` ports opencode's official SessionCompaction: when a request exceeds context limits it keeps the tail, generates an anchored summary via the zen model, and rebuilds `[system] + [summary] + [recent]`.

### Data & persistence

Everything resolves through `kit.ResolveDataPath` (`internal/kit/data.go`): `<exe>/data/<file>` preferred, fallback `<cwd>/data/`, auto-created.

- `.cline-accounts.json` (mode 0600) — accounts, rotation index, API keys, default model. Access tokens are **never persisted** (`json:"-"`, memory only, `workos:` prefixed). Mutations are pool-mutex-guarded and saved on every change.
- `.zen-config.json` — zen upstream config (key, proxies, strategy, compaction parameters).
- `zen-stats.jsonl` / `requests.jsonl` — append-only per-request stats and request logs (10MB cap → file wiped).
- `override.md` — if present in the working dir, replaces the system prompt for all requests.

## Gotchas

- `proxyConfig` (account strategy, custom headers) is **in-memory only** — lost on restart. Only `DefaultModel` persists.
- Unknown Cline models silently fall back to the default model (`normalizeRequestModel`, models.go:221); zen paid models are rejected.
- `"version": "go-1.1"` is hardcoded in `/health` and the admin config — no real version negotiation exists.
- `activeCount` is computed once at startup and goes stale (used by the health endpoint and the no-accounts guard).
- README's 项目结构 section is stale — code lives in `internal/app|cline|kit`, data in `data/`.
- Commit history is in Chinese. `capture.go` and `capture-logs/` are local-only debug tooling (gitignored, not in the repo).
