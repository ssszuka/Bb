# Arnyt Backend: HTTP, Caching, and Database Guidance

## Summary
- Keep uWebSockets.js for signaling; consider Fastify for REST to gain schema validation/plugins while keeping WS latency low. Either path should share auth/rate-limit/logging and consistent CORS/body size limits.
- Standardize on Drizzle ORM for most queries and migrations, with `drizzle.sql` for hot paths; run through PgBouncer with safe client settings.
- Use Valkey (Redis-compatible) with ioredis; prefer scripts/pipelines for atomicity and rate limiting; enforce consistent key shapes and TTLs.
- Provide operational expectations (logging/metrics/probes/backpressure) and a forward-looking OAuth layout with existing path aliases.

## Current State
- Transport: uWebSockets.js handles both WS (`/ws`) and REST; custom JSON body parsing and CORS; rate limiting via Valkey sorted-set pipeline.
- Data: PostgreSQL via `pg` pool; migrations are raw SQL under `migrations/`.
- Cache: Valkey via `ioredis`; Lua used for outbox drain; presence/rate-limit keys exist.
- Aliases (tsconfig): `@db`, `@cache`, `@jobs`, `@mw`, `@messages`, `@push-noti`, `@turn`, `@ws-handlers`, `@oauth`, etc.

## HTTP / WS Stack
- **Option A (recommended)**: uWS for WebSocket signaling; Fastify (Node 22) for REST on `127.0.0.1:<rest_port>`.
  - Reverse proxy or uWS HTTP passthrough to Fastify; keep `MAX_PAYLOAD_BYTES` and REST body limits aligned.
  - Shared middleware: JWT auth (`@mw/authGuard`), rate limit (`@mw/rateLimitRest`), request-id logging, CORS.
  - Validation: Fastify schemas (zod/TypeBox) for REST payloads; reply serialization enabled.
- **Option B (low churn)**: Keep uWS for REST; reuse body parser + CORS; add schema validation via lightweight runtime checks where needed.
- WS specifics:
  - Backpressure: drop when `getBufferedAmount > ~1MB` (already in code); log drops with request-id.
  - Ping/pong: keep server-driven pings; presence TTL slightly above ping interval.
  - Routes: maintain `/ws` only; REST should not share the same listener when Fastify is added.

## Database (PostgreSQL + PgBouncer + Drizzle)
- Client config (via PgBouncer):
  - Disable prepared statements (`statement_timeout` low; `pgbouncer` + `pg` -> set `connectionString`, `max` small, `idleTimeoutMillis` short).
  - For Drizzle `postgres-js` or `pg` driver: set `prepare` off / `statement_mode: 'simple'` if available.
- Usage pattern:
  - Default: Drizzle ORM queries for CRUD and migrations (place generated SQL under `migrations/`).
  - Critical paths: use `drizzle.sql` with parameterized raw SQL; avoid ORM abstractions in hot loops.
  - Migrations: keep SQL files; consider adding Drizzle migration metadata alongside existing `_migrations` table.
- Concurrency:
  - Pool sizing: `poolMax` small (e.g., 10–15) because PgBouncer manages fan-out; short `connectionTimeout`.
  - Transactions: keep short; wrap cache flush/DB writes atomically where possible.

## Caching / Valkey
- Client: continue `ioredis` (Valkey compatible).
- Key spaces:
  - Outbox: `outbox:{deviceId}`
  - ACKs: `acks:{sessionId}`
  - Rate limit: `rl:{scope}:{id}`
  - Presence: `presence:{deviceId}`
- Scripts:
  - Use EVALSHA for outbox drain + delete; keep BUFFER_TTL (10m) after DB flush.
  - Pipelines for rate limiting (trim window, add entry, zcard, expire) and presence set/delete.
  - Crash recovery: on startup/shutdown run outbox flush before closing DB/cache.
- TTL policies:
  - Presence TTL a bit above ping interval.
  - Rate-limit keys expire at window length.
  - ACK/outbox TTLs to prevent unbounded growth (already 10m).

## Operational Guidance
- Logging: JSON structured; request-id on every REST and WS message; redact secrets (`token/secret/key/password/fcm_token/nonce/identity_pub`).
- Metrics: capture WS connects, auth failures, backpressure drops, rate-limit hits, DB/query latency (p95/p99), cache latency.
- Probes:
  - Liveness: process up + event loop responsive.
  - Readiness: DB reachable, Valkey reachable, migrations applied; if Fastify, expose `/healthz` and `/readyz`.
- Limits:
  - Align uWS `maxPayloadLength` with REST `REST_MAX_BODY_BYTES`.
  - Enforce REST body size at Fastify level if adopted.

## Future OAuth Layout (aliases already defined)
- Paths: `src/features/oauth/{email,google/native,google/web,github,...}` with `dev.txt` placeholders; import via `@oauth/*`.
- Keep WS auth (JWT) separate from OAuth flows; OAuth issues device/user tokens that JWT strategy consumes.
- For multiple providers, share common strategy helpers under `src/features/oauth/shared/` (add when needed); continue to use aliases (`@messages`, `@push-noti`, `@turn`, `@ws-handlers`) for feature code.

## Adopt Now / Next
- **HTTP/WS**
  - Now: keep uWS WS; decide Fastify vs. uWS REST; align CORS/body limits; ensure request-id everywhere.
  - Next: if Fastify chosen, stand up internal REST port and proxy; add schema validation + response serialization.
- **Database**
  - Now: add PgBouncer-safe config (no prepared statements) and small pool; document Drizzle default + raw SQL for hot paths.
  - Next: introduce Drizzle migrations alongside existing SQL; migrate select modules to Drizzle.
- **Cache**
  - Now: codify key naming/TTL; keep Lua + pipelines; ensure startup/shutdown flush order (cache→DB then close).
  - Next: move scripts to EVALSHA with fallback; add metrics around script latency/error.
- **Ops**
  - Now: standardize logging fields/redaction; define health/readiness endpoints; track rate-limit/backpressure metrics.
  - Next: add SLOs (p95 REST latency, WS auth success rate) and alerts.

## Assumptions
- Fastify can be added alongside existing uWS without touching WS logic.
- PgBouncer fronts Postgres; Node 22 is available.
- In-repo doc is the canonical guidance (no Notion page available).
