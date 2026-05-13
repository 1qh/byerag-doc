# GOTCHAS

Evolving per-topic gotcha placeholder. Each section accrues lessons as the build hits friction. Per `book/PHILOSOPHY.md` "Capture gotchas at every milestone".

When a new gotcha lands: append one paragraph under the most relevant section (what surprised, why it surprised, what to do next time). No append-only "recent lessons" bucket — merge into topic.

## Convex self-host

- **Postgres 18+ breaks default volume mount path** — PGDATA moved to `/var/lib/postgresql/<MAJOR>/docker`. Stick to `postgres:17-alpine`. Surfaced 2026-05-14 P0 boot.
- **POSTGRES_URL must NOT include `/dbname`** — Convex backend errors `cluster url already contains db name`. Use `postgres://user:pass@host:port`; Convex derives db name from `INSTANCE_NAME` (dashes→underscores). Surfaced 2026-05-14 P0 boot.
- **`docker compose restart` ignores `.env` changes** — env substitution happens at container create time. Use `docker compose up -d --force-recreate <service>`. Surfaced 2026-05-14 P0 boot.

## Ollama embedding

- **Native Mac Ollama, NOT dockerized** — `ollama serve` runs on host; Convex container reaches via `host.docker.internal:11434`. Containerized Ollama wastes RAM + duplicates model file. Surfaced 2026-05-14 P0 boot.
- **Model slug `nomic-embed-text-v2-moe`** (dashes, single segment). NOT colon-tag form `nomic-embed-text:v2-moe` — that's wrong; non-existent tag of a different base model. Library URL is authoritative. Surfaced 2026-05-14 P0 boot when pull returned `Error: pull model manifest: file does not exist`.
- **Use OpenAI-compat `/v1/embeddings`** endpoint, NOT native `/api/embed` — portable across providers. Same model name, OpenAI-shape request/response.

## ClamAV scan

- **`clamav/clamav:latest` is amd64-only** — no arm64 manifest. Use `clamav/clamav-debian:latest` (multi-arch) on Apple Silicon. Surfaced 2026-05-14 P0 boot.

## Docker + gVisor

(none yet)

## Kimi proxy

(none yet)

## Ollama embedding

(none yet)

## ClamAV scan

(none yet)

## Agent SDK

(none yet)

## pm4ai + lintmax

(none yet)

## Next.js 16 + Turbopack

(none yet)

## Convex schema migrations

(none yet)

## Vector search filter pushdown

(none yet)

## Sandbox lifecycle (create / resume / kill)

(none yet)

## Stream pipeline (sandbox → /api/stream/event → reactive sub → client)

(none yet)

## Cross-user isolation

(none yet)

## CLI auth (device flow + PAT)

(none yet)

## Assessment tests — generation

(none yet)

## Assessment tests — review queue

(none yet)

## Assessment tests — attempts

(none yet)

## Assessment tests — assignments + re-arm

(none yet)

## Assessment tests — chat agent training tools

(none yet)

## Dashboard — top strip + cost cycle

(none yet)

## Dashboard — gradebook

(none yet)

## costRecords aggregation

(none yet)

## Agent auto-assign cron

(none yet)

## Departments

(none yet)

## Operator-host port collisions

Operator's Colima runs many concurrent compose projects: `map_*` (timescaledb / temporal / apicurio / pgbouncer / nats / kafka / minio / valkey / glitchtip / grafana / typesense), `va_*` (web on 3002, postgres), `vbfe-*` (minio on 9000/9001, nginx on 5176, backend on 5174, postgres), `noboil_*` (convex-backend on 4100/4101, dashboard on 4102, minio on 4104/4105, spacetimedb on 4200/4103, postgres-17), `k3d-truecare-pilot-local-*` (k3d cluster). byerag ports chosen to avoid all of these: admin=3001, user=3003 (not 3002 — va-web), Convex API=3210, Convex site=3211, Ollama=11434 (compose-internal). Audit `lsof -iTCP -sTCP:LISTEN` + `docker ps --format '{{.Names}}\t{{.Ports}}'` before adding any new host-bound service.
