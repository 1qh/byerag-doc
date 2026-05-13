# GOTCHAS

Evolving per-topic gotcha placeholder. Each section accrues lessons as the build hits friction. Per `book/PHILOSOPHY.md` "Capture gotchas at every milestone".

When a new gotcha lands: append one paragraph under the most relevant section (what surprised, why it surprised, what to do next time). No append-only "recent lessons" bucket — merge into topic.

## Convex self-host

(none yet)

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
