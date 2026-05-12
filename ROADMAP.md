# ROADMAP

Phase sequence by data dependency, not by stakeholder mental model. Per `book/PHILOSOPHY.md` "Single-pass build". P0 names a phase, not a priority — every phase ships in this pass.

## P0 — Substrate baseline

- Monorepo bootstrap via pm4ai.
- Convex self-host compose stack running locally (backend + Postgres + admin key generated).
- Convex schema as canonical in `SCHEMAS.md`.
- `@convex-dev/auth` wired with Google OAuth client; callback URIs registered for localhost dev ports.
- Two thin apps (`apps/admin`, `apps/user`) scaffolded with a sign-in screen + empty chat screen.

## P1 — Sandbox + proxy

- Docker + gVisor (`runsc`) runtime installed on host.
- `sandboxClient.ts` implementation using `dockerode` exposing `createSandbox` / `connectSandbox` / `commands.run` matching the substrate reference's E2B-shaped interface.
- Convex `httpAction` proxy forwarding to Kimi at `api.kimi.com`. Per-chat bearer scheme. Cost controls per `SECURITY.md`.
- Agent script embeds into sandbox at boot, scrubs secrets from env, builds proxy bearer, runs `unstable_v2_createSession` / `unstable_v2_resumeSession`, posts each event to `/api/stream/event`.

## P2 — Docs corpus

- `docs` table schema active in Convex.
- Upload endpoint accepting multipart, routing through ClamAV stateless scan service.
- Convex `_storage` blob write on scan pass.
- Doc-list + doc-read Convex actions.
- Admin app upload widget + shared docs panel.
- User app upload widget + own docs panel.

## P3 — Agent tools

- `byerag docs list` CLI command + Convex action.
- `byerag docs read` CLI + action.
- `byerag docs grep` CLI + action (iterate matching rows, regex over content text).
- `byerag docs diff` CLI + action.
- Tool registry codegen wired (`packages/cli/bin/x-codegen.ts`).
- Skill blob served at `/api/cli/skill` describes byerag CLI surface.

## P4 — Embedding + similar

- Ollama compose service running `nomic-embed-text:v2-moe`.
- Embedding computed on doc ingest (Convex action POSTing to `http://ollama:11434/api/embed`).
- `vectorIndex` on `docs.embedding`.
- `byerag docs similar` CLI + action.

## P5 — Chat UX

- Chat sidebar (thread list per user).
- Chat pane with streaming render via Convex reactive `streamEvents` subscription.
- Tool-call render (collapsible breadcrumbs).
- Citation chips linking back to doc viewer.
- Doc viewer (markdown preview, scrollable).

## P6 — Cost + audit

- Daily owner $ cap enforced in proxy.
- Per-chat turn budget.
- Rate limits (owner + chat).
- Audit log query view (admin only).

## P7 — Verify + harden

- End-to-end smoke covering: upload → scan → ingest → embed → chat → tool call → answer → citation.
- Cost-control proof: simulated burst exhausts budget, returns 402.
- Sandbox isolation proof: prompt-injection inside a doc trying to `curl` outside is blocked at network layer.
- Cross-user isolation proof: user A's docs invisible to user B's sandbox.
- ALL items in `VERIFY.md` pass.

## P8 — Polish

- Empty states, error toasts, loading skeletons.
- Doc-viewer keyboard nav.
- Chat history export.
- Backups: daily `pg_dump` cron + tested restore path.
