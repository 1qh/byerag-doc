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

- **`node:20-slim` preinstalls a `node` user at UID 1000** — `useradd -u 1000 agent` fails with `useradd: UID 1000 is not unique`. Drop the existing user first: `RUN userdel -r node 2>/dev/null || true` before `useradd -m -u 1000 -s /bin/bash agent`. Surfaced 2026-05-14 P0 sandbox image build.
- **Local dev uses plain `runc`, not `runsc`** — gVisor deferred to prod Linux per `docker-gvisor-sandbox.md`. Colima on Mac doesn't ship `runsc`; trying `--runtime=runsc` locally fails. Local hardening relies on `--cap-drop ALL --security-opt no-new-privileges` instead.
- **Sandbox runs as `User: agent` (uid 1000) — cannot write to `/usr/local/bin`** — any CLI wrapper / symlink the agent-run action places must land under a path the `agent` user owns. `/home/agent/.bun/bin/` is part of the container's default `PATH` (`/home/agent/.bun/bin:/usr/local/bin:/usr/bin:/bin`), is `agent`-owned, and is the canonical home for provider wrappers (`docs`, `training`, …) per `sandbox-image-and-cli-delivery.md`.
- **`SANDBOX_PATH` env-override does not always reach the Claude SDK Bash subshell** — the Claude Agent SDK spawns child Bash processes whose `PATH` is inherited from the container default, not always from the parent agent process's env-set `PATH`. The defense: place CLI wrappers at a path already on the container default `PATH` (`/home/agent/.bun/bin/`); don't rely on the env override propagating.
- **POSIX wrapper, not symlink, for CLI provider binaries** — a symlink at `/home/agent/.bun/bin/<provider>` pointing to `/home/agent/cli.mjs` lets the next `printf … > path` overwrite cli.mjs through the symlink. Use a 45-byte wrapper script (`#!/bin/sh\nexec node /home/agent/cli.mjs "$@"`) written via `printf` after `rm -f` of any prior file; chmod +x in the same `&&` chain.
- **Docker exec `sh -lc` is a login shell — strips the agent process's PATH** — `sh -l` re-sources `/etc/profile` which sets `PATH=/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games`, overriding the `PATH` env passed via Docker exec's `Env` field. The agent process loses `/home/agent/.bun/bin` from PATH; the Claude SDK's Bash subshells then can't find the provider wrappers. Canonical: `sh -c` (non-login) for exec invocations — preserves the Env-set PATH end-to-end. Captured in `sandboxClient.ts:execInside`.

## Kimi proxy

- **`stop_reason="tool_use"` after the last tool call leaves the result empty** — Kimi sometimes ends an agent turn with a tool_use block and never emits the follow-up text turn that would synthesize the answer; the SDK records `result.result = ""` and the chat shows no answer body. Mitigations baked into `system-prompts.md`: explicit "final-answer protocol" mandating a plain-text response after the tool chain; smoke harness asserts non-empty assistant text + keyword + citation match per scenario.
- **Stream-event firehose exceeds default rate-limit caps** — Kimi emits per-token deltas during streaming; an agent that composes 4+ tool calls in one turn produces 1K–8K stream events. Default per-chat cap (300/min) and per-owner cap (900/min) starved the chat of events and corrupted `complete`-time message reconstruction. Canonical caps for `/api/stream/event`: per-chat 8_000, per-owner 20_000, refilled across a 60s window. Captured in `SECURITY.md` rate-caps table.

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

- **Matryoshka shorter-dim queries against a fixed-dim index**: Convex `vectorIndex` declares `dimensions: 768` and accepts only that exact length. Matryoshka prefix queries (256 / 512) are realized by truncating the query vector to the first N dims and zero-padding the remainder back to 768. Cosine over the zero-padded query equals dot-product over the first N dims of the stored vector (zero positions contribute 0), giving the canonical Matryoshka semantics without re-indexing. Top-score increases monotonically with dim because more signal is included; rank is preserved. Helper: `matryoshkaTruncate(vec, dim)` in `apps/backend/convex/docsEmbed.ts`; consumed by `tools/docs/similar.ts`.

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
