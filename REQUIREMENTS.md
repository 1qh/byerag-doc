# REQUIREMENTS

Feature catalog. Canonical state, not transition. Each row is required for the launch-tier endpoint per `book/PHILOSOPHY.md`.

## Auth

- Google OAuth sign-in on both apps via `@convex-dev/auth`.
- Role determined by which app the OAuth callback originated from (admin app → admin role; user app → user role).
- Session persisted in Convex `authTables`.
- Sign-out clears session cookie.

## Upload pipeline (admin and user)

Every upload runs through these gates in order; failure at any gate ends the flow with the matching UI:

1. **Scan** (ClamAV) — malicious → `scanStatus='quarantined'`, no blob; toast: `⚠️ Your file was rejected because it appeared suspicious. Reason: <signature>.` Audit log records.
2. **Dedup** (sha256 in same scope) — match → no new row; toast: `this file is already in your library (uploaded as <filename> on <date>).`
3. **Version-conflict** (same filename in same scope, different content) — blocking modal: `Replace · Keep both · Cancel`. Server applies per `upload-dedup-and-version-prompt.md`.
4. **Policy classifier** (LLM-backed; per `policy-relevance-classifier.md`) — rejected → `policyStatus='rejected'`, blob retained for admin review; toast: `This file is rejected as not matching our policy. Reason: <reason>.` + button: `Request review`.
5. **Extract + chunk + embed** — produces `extractedText`, `lang`, `docChunks`, `docs.embedding`.
6. **Clean insert** → doc becomes searchable. Admin upload: `scope='shared'`, `owner=null`. User upload: `scope='mine'`, `owner=<user email>`.

## Cross-scope rule

Dedup + version-conflict checks run within the same `(scope, owner)` partition. A shared-scope doc with content X does NOT block a user uploading content X to `mine` (and vice versa).

## Admin-only surfaces

- **Policy editor** at `/admin/policy` — large textarea + Save. Every save logged.
- **Quarantine queue** at `/admin/quarantine` — paginated list of `policyStatus='rejected'` docs. Per row: Approve (override → `approved`, `policyOverriddenBy=<admin>`) · Confirm reject (purge blob + chunks; row retained for audit).
- **Review-request queue** — docs where user clicked `Request review`; sorted by request time. Same approve/confirm-reject actions.

## Chat (both apps)

- User starts a new chat or resumes an existing one from a sidebar.
- Each chat row is `(owner, app)` scoped; visibility is per-user, never cross-user.
- User types a question; the agent reads the question, calls docs tools as needed, streams an answer with citations.
- Answer is rendered with tool-call breadcrumbs (collapsible) and citations linking back to source docs.

## Docs CLI surface (agent-facing)

Listed canonically in `CLI-SURFACE.md`. Includes at minimum:

- `byerag docs list --scope shared|mine|both`
- `byerag docs read --id <docId>`
- `byerag docs grep --pattern <regex> --scope <s> [--limit N]`
- `byerag docs diff --a <docIdA> --b <docIdB>`
- `byerag docs similar --query <text> --scope <s> [--limit N] [--dim 256|512|768]`

## Sandbox

- Per-owner Docker + gVisor container, created on first chat message, resumed across subsequent messages.
- Container has read access to a synced doc scratch dir (`/workspace/shared` + `/workspace/mine/<owner>`) mirroring the user's visible scope.
- Network isolated; only Convex internal endpoint reachable, nothing else.
- LLM API host is reachable only from Convex (the proxy), never from the sandbox.
- Container pauses between user messages; agent process re-spawns or resumes via SDK session id.

## Proxy

- Convex `httpAction` proxies sandbox LLM requests to Kimi.
- Per-chat bearer (`sk-ant-oat01-proxy_<chatId>_<secret>`) authenticates the sandbox to the proxy.
- Proxy swaps bearer for the real Kimi key before forwarding.
- Proxy enforces per-owner daily $ cap, per-chat turn budget, per-chat rate limit, per-owner rate limit, upstream-path allowlist, max-body cap.
- Proxy reserves budget pre-call (estimate worst-case), settles post-call (actual usage).

## Embedding

- On doc upload: text extracted (`pdftotext`, `pandoc`, etc.) → first `min(8K, content)` chars sent to local Ollama at `/api/embed` w/ `model="nomic-embed-text:v2-moe"`, prefix `search_document: `.
- Embedding stored as `docs.embedding` field, indexed via `vectorIndex` with `dimensions=768`, filter fields `owner`, `scope`.
- On `docs similar` tool call: query text prefixed `search_query: `, embedded, `ctx.vectorSearch` with ACL filter pushdown.

## Audit

- Every CLI exec call logged to `auditLogs` (owner, command, args summary, ok, mode).
- Every uploaded doc immutable; soft-delete supported but not exposed P0.

## Cost control

- Per-owner daily $ cap (configurable, default $5/owner/day).
- Per-chat turn budget (default 50 proxy calls / chat).
- Per-chat rate limit (default 300 LLM calls / minute).
- Per-owner rate limit (default 600 LLM calls / minute / owner).

## Operations

- Egress firewall on the host blocks all outbound except the LLM API host (`api.kimi.com:443`).
- Container `--network none` enforces sandbox-side isolation.
- Backups via daily `pg_dump` of Convex's backing Postgres.
