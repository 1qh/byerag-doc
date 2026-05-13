# local-dev-loop

`bun dev` runs the full local dev stack from the repo root. Per-app ports are fixed in `package.json` so OAuth callback URIs match.

## Ports

- admin app: `3001`
- user app: `3002`
- Convex cloud API: `3210` (compose-mapped)
- Convex site URL: `3211` (compose-mapped)
- Ollama: `11434` (compose-mapped)
- Scan service: `8080` (compose-internal only — no host mapping)

## Commands (root `package.json`)

- `bun dev` — `docker compose up -d` + `turbo run dev` (parallel: admin, user, convex codegen watch)
- `bun dev:stack` — only `docker compose up -d` (compose stack only, no Next dev servers)
- `bun dev:apps` — only Next dev servers (assumes compose stack already up)
- `bun sync` — `apps/backend/sync.ts` reads `.env` → pushes to Convex env
- `bun smoke:agent` — end-to-end smoke against the local stack
- `bun fix` — lintmax fix
- `bun test` — bun test across workspaces

## First-time setup

1. `cp apps/backend/.env.example apps/backend/.env`
2. Fill in `AUTH_GOOGLE_ID`, `AUTH_GOOGLE_SECRET`, `KIMI_API_KEY`, `BACKUP_AGE_PUBKEY` (operator-supplied).
3. `bun i`
4. `bun dev:stack` — compose boots; on first boot generates `CONVEX_SELF_HOSTED_ADMIN_KEY` + `JWT_PRIVATE_KEY` + `JWKS` into `.env` automatically.
5. `bun sync`
6. `bun dev:apps`

## Hot reload

- Convex functions: `bunx convex dev` watches `apps/backend/convex/**/*.ts`, pushes changes on save (built into `turbo run dev` graph).
- Next.js (admin, user): Turbopack hot reload on `.ts`/`.tsx` save.
- Shared packages (`packages/react`, `packages/cli`): bun workspaces resolves to source; Next pulls fresh on save (no rebuild step).
- Sandbox image: NOT hot-reloaded. Image rebuild required on Dockerfile change; on tool registry change, only the embedded `CLI_SCRIPT` const regenerates (no image rebuild needed because CLI is injected at agent-run time per `sandbox-image-and-cli-delivery.md`).

## Beats

- **One terminal running everything**: log streams interleave; debugging hard.
- **No compose, run each service via OS package manager**: drift between dev and prod; "works on my machine".
- **Single-port dev (admin + user behind one Next instance)**: breaks the apps-as-separate-origins shape; complicates cookie scoping + route guards.

## Real cost

- Compose stack uses ~2 GB RAM idle (Postgres + Convex + Ollama + ClamAV + scan service).
- First boot pulls Ollama model (~1 GB download); subsequent boots cached.

## Gotcha for Claude

- The OAuth client's authorized callback URIs MUST include `http://localhost:3001/api/auth/callback/google` AND `http://localhost:3002/api/auth/callback/google` AND Convex site URL on port 3211. Missing any → `redirect_uri_mismatch`.
- `127.0.0.1` and `localhost` are different origins for OAuth; register both.
- `bun dev:stack` checks compose health and exits non-zero if any service unhealthy after 60s.
- `bun smoke:agent` requires a clean schema; runs `bun run reset:db` first (drops + re-applies V001). Pre-launch only — never on a DB that holds data per `book/HARD-RULES.md` "Pre-launch: one migration only".
