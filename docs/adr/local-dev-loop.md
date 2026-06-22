# local-dev-loop

`bun dev` runs the full local dev stack from the repo root. Per-app ports are fixed in `package.json` so OAuth callback URIs match. Ports chosen to avoid collision with operator's existing services.

## Ports

- admin app: `3001`
- user app: `3003`
- Convex cloud API: `3210` (compose-mapped, `127.0.0.1` only)
- Convex site URL: `3211` (compose-mapped, `127.0.0.1` only)
- Ollama: `11434` (compose-internal — no host mapping)
- ClamAV `clamd`: `3310` (compose-internal — no host mapping)
- Internal Postgres: `5432` (compose-internal — no host mapping)

Project namespace `byerag` in `compose.yml` prefixes all containers / volumes / networks. No collision with operator's existing `map_*` / `va_*` / `noboil_*` / `vbfe-*` / `k3d-*` projects.

## Commands (root `package.json`)

- `bun dev` — `docker compose up -d` + `turbo run dev` (parallel: admin, user, convex codegen watch).
- `bun dev:stack` — `docker compose up -d` only.
- `bun dev:apps` — Next dev servers only (assumes compose up).
- `bun sync` — `apps/backend/sync.ts` reads `.env` → pushes to Convex env.
- `bun smoke:agent` — end-to-end smoke against the local stack.
- `bun fix` — lintmax fix.
- `bun test` — bun test across workspaces.

## First-time setup

1. `cp apps/backend/.env.example apps/backend/.env`
2. Fill in `AUTH_GOOGLE_ID`, `AUTH_GOOGLE_SECRET`, `KIMI_API_KEY`, `BOOTSTRAP_ADMIN_EMAIL` (operator-supplied). `BACKUP_AGE_PUBKEY` agent generates via `age-keygen` if absent.
3. `bun i`
4. `bun dev:stack` — compose boots; on first boot generates `CONVEX_SELF_HOSTED_ADMIN_KEY` + `JWT_PRIVATE_KEY` + `JWKS` into `.env` automatically; seeds default `corpus_policy` + `agent_auto_assign_enabled='false'` settings rows.
5. `bun sync`
6. `bun dev:apps`

## Hot reload

- Convex functions: `bunx convex dev` watches `apps/backend/convex/**/*.ts`; pushes on save (in `turbo run dev` graph).
- Next.js (admin, user): Turbopack hot reload on `.ts` / `.tsx` save.
- Shared packages (`packages/react`, `packages/cli`): bun workspaces resolves source; Next pulls fresh on save.
- Sandbox image: NOT hot-reloaded. Rebuild on `Dockerfile` change. Tool registry change re-generates embedded `CLI_SCRIPT` const (no image rebuild — CLI injected at agent-run per `sandbox-image-and-cli-delivery.md`).

## Mac + Colima posture

Operator runs macOS 26 arm64 + Colima (`macOS Virtualization.Framework`). Docker daemon lives in Colima's Linux VM (aarch64). Sandbox containers run inside that VM. gVisor (`runsc`) deferred to prod Linux server — local dev uses plain `runc` w/ `--cap-drop ALL --security-opt no-new-privileges` for hardening parity (per `docker-gvisor-sandbox.md`).

Colima default resources: 4 CPU / 24 GiB RAM / 100 GiB disk — sufficient for full stack.

## Beats

- **One terminal running everything**: log streams interleave; debugging hard.
- **No compose, OS-package manager per service**: drift between dev and prod; "works on my machine".
- **Single-port dev (admin + user behind one Next instance)**: breaks the apps-as-separate-origins shape; complicates cookie scoping + route guards.

## Real cost

- Compose stack uses ~3-5 GB RAM idle (Postgres + Convex + Ollama + ClamAV).
- First boot pulls Ollama model (~1 GB download); subsequent boots cached.

## Gotcha for Claude

- OAuth client's authorized callback URIs MUST include: `http://localhost:3001/api/auth/callback/google`, `http://localhost:3003/api/auth/callback/google`, AND `http://localhost:3211/.well-known/openid-configuration/callback/google`. Missing any → `redirect_uri_mismatch`.
- `127.0.0.1` and `localhost` are different origins for OAuth; register both if used interchangeably.
- `bun dev:stack` checks compose health; exits non-zero if any service unhealthy after 60s.
- `bun smoke:agent` requires a clean schema; runs `bun run reset:db` first (drops + re-applies V001). Pre-launch only — never on a DB that holds data per `book/HARD-RULES.md` "Pre-launch: one migration only".
- Operator has many concurrent compose projects on Colima (`map_*`, `va_*`, `noboil_*`, `vbfe-*`, k3d cluster). byerag uses `name: byerag` to namespace; volumes / networks / containers all `byerag_*` prefixed. Audit ports + names before any new service add — collision check is mandatory.
- Local-dev agent auth bypass: on first compose boot, seed script inserts a `cliTokens` row with `source='dev'` mapped to `BOOTSTRAP_ADMIN_EMAIL[0]`; plaintext token written to `apps/backend/.env.local` as `DEV_CLI_TOKEN=<token>`. Production deployments never seed this.

## MUST

- Set every `.env` Convex URL (`NEXT_PUBLIC_CONVEX_URL`, `CONVEX_SELF_HOSTED_URL`, `CONVEX_SITE_URL`) to literal `127.0.0.1`, never `localhost`. Why: macOS `localhost` resolves `::1` first but Convex binds `127.0.0.1:3210` (v4) → ECONNREFUSED.
- Forward all 4 SSH dev ports on remote dev machines: `3001` admin, `3003` user, `3210` Convex API, `3211` Convex site. Why: missing 3210/3211 fails browser WebSocket sync silently — login screen sits "Loading…".
- Give `apps/backend` a `dev` script `bun run build-agent && bun run codegen && bunx convex dev` (persistent watch). Why: a missing `dev` script leaves `turbo dev` cold — every push is a fresh `convex dev --once` that re-analyzes.
- Run deploy single + sequential with the box quiet and no users active. Why: concurrent/parallel pushes saturate the single isolate pool and starve runtime UDFs (`auth:signIn` hits the 1s UDF budget, sign-in breaks).
- Free host CPU before deploy so the V8 module-analyze keeps CPU and the node-executor bundle is warm. Why: the 4s analyze isolate timeout is NOT configurable; `start_push` fails `400 InvalidModules: Function execution timed out (maximum duration: 4s)` under host saturation.

## NEVER

- Loop-spam `convex dev --once` or deploy concurrently with app use. Cost: repeated/parallel pushes saturate the isolate pool, `auth:signIn` hits the 1s UDF budget, sign-in breaks.

## Pitfall

- SSH port-forwards forward IPv4 only, so the same `::1`-vs-`127.0.0.1` trap recurs on remote dev machines — keep all URLs on `127.0.0.1`.
- Self-host deploy is environment-fragile: a reliable green deploy needs both a warm node bundle and CPU headroom; co-tenant containers + host procs driving host load high cause the 4s analyze timeout, and the deploy goes green in ~40s once the box is freed.
