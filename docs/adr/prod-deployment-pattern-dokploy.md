# prod-deployment-pattern-dokploy

Production deployment pattern for byerag: Dokploy on operator-owned VM, mirroring the substrate-reference (claude2b) shape. Local dev = Colima compose on operator's Mac (per `local-dev-loop.md`). Prod = Dokploy compose on a Linux VM. Same compose file shape; differences are TLS termination, reverse-proxy routing, and managed env injection.

## Mechanism

Dokploy is the operator's self-hosted PaaS (admin URL + API key stored in agent memory, never tracked). Each compose stack registered as a Dokploy "compose" resource. Two source modes:

- **`sourceType: raw`** — compose YAML pasted into Dokploy's UI. Manual edits via web UI; no git wire.
- **`sourceType: github`** — compose YAML pulled from a git repo branch. `autoDeploy: true` re-deploys on push to the watched branch.

For byerag prod: `sourceType: github` pointing at this code repo's `compose.yml`. Push to `main` → Dokploy pulls + redeploys.

## Network shape

- Each Dokploy compose joins the external `dokploy-network` bridge (Traefik routes traffic there).
- Traefik terminates TLS + maps `<host>.<your-domain>` → service port.
- Internal compose-only services (Ollama, ClamAV, Postgres) do NOT join `dokploy-network` — they stay on the compose's `default` network.

## Env injection

Sensitive env vars (DB_PASSWORD, INSTANCE_SECRET, KIMI_API_KEY, AUTH_GOOGLE_ID, AUTH_GOOGLE_SECRET, BOOTSTRAP_ADMIN_EMAIL, BACKUP_AGE_PUBKEY) live in Dokploy's per-compose env panel. NOT in the git-tracked `compose.yml`. The compose file references them via `${VAR_NAME}` expansion.

`apps/backend/sync.ts` continues to push secrets to Convex env on first boot in prod (same script as local dev).

## TLS

Traefik handles TLS via Let's Encrypt. Convex backend runs with `DO_NOT_REQUIRE_SSL=1` because Traefik already terminated TLS — backend sees plain HTTP from the local Docker bridge.

## Domain mapping (operator chooses literal hosts)

Reference shape from substrate's prod deployment:

| Service | Internal port | Public host (operator-chosen) |
|---|---|---|
| Convex API | 3210 | e.g. `cvex-api.<domain>` |
| Convex site (HTTP actions) | 3211 | e.g. `cvex-backend.<domain>` |
| Convex dashboard | 6791 | e.g. `cvex-dashboard.<domain>` |
| Admin Next.js app | 3001 | e.g. `admin.<domain>` |
| User Next.js app | 3003 | e.g. `app.<domain>` |

Operator registers DNS records (or CNAME to Dokploy host); Traefik picks up routing.

## Healthchecks

- Convex backend: `curl -f http://localhost:3210/version` (5s interval, 5s start period).
- Postgres: `pg_isready -U postgres`.
- Ollama: `ollama list`.
- ClamAV: `clamdcheck.sh`.

## Deploy lifecycle

- **No agent-side manual deploys.** Never run `bunx convex deploy`, `vercel deploy`, `bun run deploy`. Git push triggers Dokploy auto-deploy.
- **Founder owns `git push origin main`** for repos with auto-deploy wired. Byerag push policy: founder authorized agent push to byerag + byerag-doc (per current session).
- **Dokploy API calls are read-only from agent.** `compose.one`, `project.all`, `application.one` for inspection. NEVER `compose.deploy`, `compose.redeploy`, `compose.start`, `compose.stop`, `compose.delete` — those are operator actions in Dokploy UI.

## Known Dokploy quirks

- **Compose-idle-on-deploy bug**: some composes accept `compose.deploy` returning success, but `composeStatus` stays `idle` and no deployment runs. Workaround: clone a working compose's shape exactly. Don't try to invoke deploy programmatically — use Dokploy UI for manual trigger on stuck composes.
- **`compose.start` returns `spawn /bin/sh ENOENT`** on some composes. Same opaque-bug class; Dokploy quirk.
- Healthcheck `start_period` < container actual start time → marked unhealthy temporarily; Dokploy may mark deploy failed even when service eventually starts. Pad `start_period` generously.

## Beats

- **Direct Compose-on-VM without orchestrator**: loses TLS automation, env injection UX, deploy-on-push, healthcheck dashboard. Dokploy gives all four free.
- **K8s on VM**: orders of magnitude more ops surface for a single-host byerag deploy.
- **Vercel for Next.js apps**: provider-managed compute without local-runnable equivalent (per book `PHILOSOPHY.md` self-host-first + local-first hostability invariant). Local dev compose can't reproduce Vercel's edge functions. Drop.

## Real cost

- One Dokploy install on a VM (operator-side ops).
- Dokploy UI clicks to set env per compose.
- Compose YAML lives in git; matches local dev shape; one source of truth.

## Gotcha for Claude

- Local `compose.yml` and prod `compose.yml` are the **same file** in byerag's code repo. Differences (host port mappings, healthchecks, env interpolation) handled via compose-override files if needed. Prefer parameterization over branching.
- Local dev binds ports to `127.0.0.1` only (`compose.yml` already does this). Prod doesn't bind to host ports at all — Traefik routes via the docker network. Override port-binding away in prod compose if needed.
- Image `ghcr.io/get-convex/convex-backend:latest` matches both environments. Pin to a digest at artifact-boundary per `book/HARD-RULES.md` "Reproducibility at artifact boundaries" (operator's choice when prod cuts a release tag).
- Future operator action: register Dokploy compose for byerag, set env panel, point `sourceType: github` at this repo's `compose.yml`, configure auto-deploy on main branch.

## Prod-built one-command bring-up

### MUST
- Bring up the full prod stack via `bun run prod` at the repo root (script `prod.sh`, wired as `"prod"` in root `package.json`). Why: one command chains the entire stack in dependency order.
- Stop the Docker stack with `docker compose down` after Ctrl-C. Why: Ctrl-C reaps only the two Next servers via a TERM trap; the Docker stack stays up.

### Pitfall
- `bun run prod` chains: `docker compose up -d --wait` (postgres + convex-backend + clamav) → Ollama healthcheck on `:11434` → `convex deploy` from `apps/backend` → `turbo build` (admin + user + packages) → `next start` for admin `:3001` + user `:3003` in parallel.
- The two Next-server PIDs go into `.cache/prod/pids`; logs into `.cache/prod/*.log`; `.cache/` is gitignored. Each step exits non-zero with the relevant log tail on failure.

## Operator-host port collisions

### MUST
- Audit `lsof -iTCP -sTCP:LISTEN` + `docker ps --format '{{.Names}}\t{{.Ports}}'` before adding any new host-bound service. Why: the operator's Colima runs many concurrent compose projects whose ports must be avoided.

### Pitfall
- byerag host ports are chosen to avoid all co-tenant projects: admin=3001, user=3003 (not 3002 — `va_*` web), Convex API=3210, Convex site=3211, Ollama=11434 (compose-internal).
- Co-tenant projects in use: `map_*` (timescaledb / temporal / apicurio / pgbouncer / nats / kafka / minio / valkey / glitchtip / grafana / typesense), `va_*` (web on 3002, postgres), `vbfe-*` (minio on 9000/9001, nginx on 5176, backend on 5174, postgres), `noboil_*` (convex-backend on 4100/4101, dashboard on 4102, minio on 4104/4105, spacetimedb on 4200/4103, postgres-17), `k3d-truecare-pilot-local-*`.
- Concrete Dokploy URL + API key + project/compose ids live in agent memory at `~/.claude/projects/-Users-o-codoc/memory/`, never in tracked docs.
