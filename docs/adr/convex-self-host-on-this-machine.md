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
