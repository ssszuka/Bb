**Title**: Architecture guidance doc (HTTP stack, caching, PostgreSQL/Drizzle) for Arnyt

**Summary**
- Create a concise Markdown doc that captures recommended choices and rationale for HTTP/WS transport, cache scripting, and database access with PgBouncer and Drizzle (raw SQL on hot paths).
- Document current state (uWebSockets.js for WS + REST, pg/ioredis direct use), call out gaps, and lay out a forward path without changing runtime yet.
- Note that no Notion page was available via MCP; the guidance will live in-repo.

**Key Content to include**
- **HTTP/WS stack**: Keep uWebSockets.js for WebSocket signaling; move REST to Fastify (Node 22) for validation/plugins, or stay on uWS REST only if minimizing churn. Describe side‑by‑side deployment (Fastify on 127.0.0.1:<rest>, uWS WS on /ws), shared middleware (auth/rate‑limit/logging), and CORS/body limits.
- **Database**: Default to Drizzle ORM for readability/migrations; raw `drizzle.sql` for critical paths. Show connection config when fronted by PgBouncer (disable prepared statements, keep low `poolMax`, short timeouts). Mention migrations location and how to phase Drizzle in alongside existing SQL.
- **Caching / Valkey**: Keep Valkey with `ioredis` (or valkey-compatible client). Document script strategy: EVALSHA for outbox drain/ack, pipelines for rate-limit and presence, consistent key naming (`outbox:*`, `presence:*`, `rl:*`), TTL policies, and crash-recovery flush order.
- **Operational guidance**: Logging/metrics expectations (per-request IDs, slow query thresholds), health/readiness probes for Fastify/uWS, and backpressure handling on WS.
- **Future OAuth layout**: Describe intended folder map (already scaffolded) for email, Google native/web, GitHub, etc., and how aliases (@oauth/*, @messages/*, @push-noti/*, @turn/*, @ws-handlers/*) should be used going forward.

**Placement & format**
- Add the doc at `docs/architecture/http-cache-db.md` (Markdown, repo-local); no code changes required.
- Include short “Adopt now / Next” checklists for each area to make the transition actionable.

**Test/Review plan**
- Manual review: verify all paths/aliases referenced match tsconfig.
- Sanity checklist: doc covers decisions, tradeoffs, and concrete next steps; no contradictory guidance with current code.

**Assumptions**
- It’s acceptable to introduce Fastify alongside the existing uWS WS server without immediate rewrite of WS logic.
- PgBouncer is or will be available in front of Postgres.
- No Notion page was accessible via MCP; in-repo doc is acceptable as the source of truth.

