# convex-self-host-on-this-machine

Convex backend runs as a local Docker compose stack on the operator's machine. Backing Postgres lives in the same compose project.

## Beats

- **Convex managed (convex.dev)**: vendor lock-in for state. byerag's non-goal: only outbound is LLM API host.
- **Plain Postgres + own backend (FastAPI / Express)**: reactive WebSocket subscriptions become a project; `scheduler.runAfter` becomes a worker queue; httpAction becomes API routes. Substrate reference proved Convex's primitives shave weeks.
- **Supabase self-host**: realtime layer present, but heavier (PostgREST + GoTrue + Realtime + Storage as separate services). Convex bundles those in one process.

## Real cost

- Convex self-host is newer than convex.dev managed. Operational maturity lower; some features lag the managed version.
- TypeScript-flavored schema (`v.*` / `defineTable`) is portable to nothing else. If a future migration off Convex happens, schema is rewritten.
- Self-host admin key generated on first compose boot; lives in operator-local `.env`. Lose it = lose admin access.

## Gotcha for Claude

- `process.env.NODE_ENV === 'production'` always inside Convex runtime regardless of dev/prod. Branch on `CONVEX_SELF_HOSTED_URL` or a custom `ALLOW_OVERRIDES` env, never on `NODE_ENV`.
- `@convex-dev/auth` requires `JWT_PRIVATE_KEY` + `JWKS` env vars; generated on first sign-in if missing, but must be persisted to `.env` so backend + source-of-truth stay aligned.
- `SITE_URL` is comma-separated; multi-origin (prod + localhost + 127.0.0.1) needs every origin listed.
- Convex env vars are pushed by `apps/backend/sync.ts` reading `.env`; never `convex env set` literals from shell — they get overwritten on next `bun run sync`.
- Backups via `pg_dump` of the compose-managed Postgres volume. Restore by replaying the dump into a fresh compose stack.

## MUST

- Keep `extractedText` doc values under Convex's 1 MiB single-value cap by writing full text to a `_storage` text blob (`docs.extractedTextStorageId`) and keeping only a bounded prefix (`EXTRACT_INLINE_MAX_CHARS`) inline. Why: inline 3.25 MiB throws `Value is too large (3.25 MiB > maximum size 1 MiB)`, stalling ingestion forever.
- Cover all prefix-only readers within `EXTRACT_INLINE_MAX_CHARS` (classify ≤4K, gen ≤12K, conflict ≤50K, snippets); chunking/embedding reads the blob. Why: vector / `docs similar` must cover the whole doc, not just the prefix.
- Drop per-stream-event `incrementEventCount` patches on the single `chatRuntime` row; track a monotonic stream `seq` against `STREAM_EVENT_HARD_CAP` instead. Why: per-event patches on one hot row cause OCC "Documents read/written too many times".
- Remove per-event stream rate-limit checks and wrap `checkRateLimit` in `safeRateLimit` that catches OCC errors and returns `true` (fail-open). Why: per-event `checkRateLimit` against the owner row contends on OCC mid-stream; rate-limit fail-open is acceptable for the stream firehose.
- Use indexed reads in ratelimit-like hot paths: `docs.countRecentQuarantines` queries `by_sha256_scope_owner` `.take(50)` then filters in JS. Why: a full-table scan over all `docs` rows times out at the 1s budget.
- Re-run `bun sync` after clearing corrupt `JWT_PRIVATE_KEY=` + `JWKS=` lines to regenerate a matched pair. Why: `sync.ts` JWT regen can write an unquoted multi-line PEM, making `.env` unparseable and breaking compose + push.

## NEVER

- Cold-restart `convex-backend` or run `docker compose down -v` casually, and never use `docker compose restart convex-backend` as recovery. Cost: cold-wipes the warm node-executor bundle (60s cold path), wipes all deployed functions (`auth:signIn` → nobody signs in), and clears `costRecords`/auth tables.

## Pitfall

- A DB wipe or JWT regen invalidates the browser cookie, so every identity-gated query returns `null`/`[]` — docs won't preview, `/training` empty, "Start does nothing", sign-in timeouts are all the SAME stale-session bug. Recovery: sign out, clear site data for the origin, sign in fresh (or use env-gated dev sign-in).
- `EXTRACT_INLINE_MAX_CHARS` readers that use `defineQuery` (`docs read`/`grep`/`diff`) cannot `ctx.storage.get` bytes, so they operate on the inline prefix; full-doc bytes require blob-reading actions; semantic search is unaffected.
