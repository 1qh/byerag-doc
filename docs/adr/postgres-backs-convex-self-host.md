# postgres-backs-convex-self-host

Postgres is Convex self-host's mandatory durable storage backend. byerag's `compose.yml` runs `postgres:17-alpine` alongside `convex-backend:latest`. Apps NEVER talk to Postgres directly; only Convex's runtime reads/writes there.

## Layer cake

```
admin / user apps · sandbox CLI
     ↓ (Convex client)
Convex backend (runtime + reactive WS + scheduler + httpAction + _storage)
     ↓ (Postgres protocol)
Postgres 17 (durable storage)
```

All byerag's schema tables (chats, messages, streamEvents, docs, docChunks, testQuestions, testAttempts, costRecords, userProfiles, etc.) live as rows in Postgres tables that Convex's runtime owns + manages. Vector indexes, search indexes, scheduled functions, blob storage metadata — all in Postgres.

## Why postgres

- Per upstream Convex self-host docs, Postgres is the only supported backend for production self-host. Mandatory, not optional.
- Matches substrate (claude2b) prod compose pattern.
- Standard tool, mature, reproducible backups via `pg_dump`.

## Version pin

`postgres:17-alpine`. Pinned major version 17 because Postgres 18+ changed the PGDATA layout (`/var/lib/postgresql/<MAJOR>/docker`), and Convex backend's image is built against postgres 17's layout. Upgrading to postgres 18 requires either:

- (a) Convex backend image rebuilt for the new layout (upstream concern; track GitHub).
- (b) Adjust compose volume mount + perform pg_dump + restore migration on upgrade.

For v0 stay on 17. Revisit when Convex upstream supports 18.

## Compose env

`compose.yml` runs postgres with:

```
POSTGRES_USER: convex
POSTGRES_DB: byerag_self_hosted    # INSTANCE_NAME with dashes→underscores
POSTGRES_PASSWORD: ${DB_PASSWORD}   # from root .env
```

Connection string for Convex backend: `postgres://convex:${DB_PASSWORD}@postgres:5432` (no `/dbname` suffix — Convex backend appends it from `INSTANCE_NAME`).

## Volumes

- `postgres-data:/var/lib/postgresql/data` — durable. Persists across `docker compose down` (only nuked by `docker compose down -v`).
- Backup via `pg_dump` from inside container, age-encrypted to operator's backup disk (per `backups-pg-dump-and-restore-drill.md`).

## Beats

- **Convex managed cloud**: vendor lock-in for state; violates self-host-only-LLM-outbound rule.
- **Embedded SQLite in Convex backend**: not supported by Convex self-host; would need fork.
- **Cockroach / Yugabyte / etc.**: not officially supported; Postgres is the only path.

## Real cost

- One postgres container (~50MB image, modest RAM idle).
- DB password as compose env var (loaded from root `.env`).

## Gotcha for Claude

- **Postgres 18+ breaks the default volume mount path** (PGDATA lives at `/var/lib/postgresql/<MAJOR>/docker`). Stick to `postgres:17-alpine` until Convex upstream supports 18.
- **POSTGRES_URL must NOT include `/dbname`** — Convex backend errors with `cluster url already contains db name`. Connection string is `postgres://user:pass@host:port` only; Convex computes db name from `INSTANCE_NAME` (replacing `-` with `_`).
- **`docker compose restart` does NOT re-read `.env`** — environment substitution happens at container create time. Use `docker compose up -d --force-recreate <service>` after env changes.
- **byerag's DB name = `byerag_self_hosted`** (computed from `INSTANCE_NAME=byerag-self-hosted`). Match POSTGRES_DB in compose.
- **Reset via `docker compose down -v` wipes the volume.** Pre-launch (unlimited rework per book PHILOSOPHY) this is the right move when schema changes; post-launch this DELETES ALL DATA.
- **Apps NEVER connect to Postgres directly.** All access through Convex backend. There's no scenario where byerag code opens a Postgres connection.
