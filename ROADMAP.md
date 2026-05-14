# ROADMAP

Phase sequence by data dependency, not by stakeholder mental model. Per `book/PHILOSOPHY.md` "Single-pass build". P0 names a phase, not a priority — every phase ships in this pass.

## P0 — Substrate baseline

- Monorepo bootstrap via pm4ai.
- `compose.yml` w/ postgres + convex-backend + ollama + ollama-init + clamav + scan service. Healthchecks per service. Bridges `internal` + `sandbox-egress`.
- Convex self-host compose stack running locally; admin key + JWT keypair generated to `.env`.
- Convex schema as canonical in `SCHEMAS.md` (includes `docs`, `docChunks`, `userContexts` and all auxiliary tables).
- `@convex-dev/auth` wired with Google OAuth client; callback URIs registered for localhost dev ports (3001 admin, 3003 user, 3211 Convex site).
- Two thin apps (`apps/admin`, `apps/user`) scaffolded with a sign-in screen + empty chat screen.
- Lefthook pre-commit + universal CI from pm4ai.
- Sandbox image built (`apps/backend/sandbox/Dockerfile`) tagged `byerag-sandbox:latest`.
- iptables / nftables rules applied per `network-bridge-rules.md`.

## P1 — Sandbox + proxy

- Docker + gVisor (`runsc`) runtime installed on host.
- `sandboxClient.ts` implementation using `dockerode` exposing `createSandbox` / `connectSandbox` / `commands.run` matching the substrate reference's E2B-shaped interface.
- Convex `httpAction` proxy forwarding to Kimi at `api.kimi.com`. Per-chat bearer scheme. Cost controls per `SECURITY.md`.
- Agent script embeds into sandbox at boot, scrubs secrets from env, builds proxy bearer, runs `unstable_v2_createSession` / `unstable_v2_resumeSession`, posts each event to `/api/stream/event`.

## P2 — Docs corpus

- `docs` + `docChunks` tables active in Convex.
- Upload endpoint accepting multipart, routing through ClamAV stateless scan service.
- Convex `_storage` blob write on scan pass.
- Doc-list + doc-read Convex actions.
- Admin app upload widget + shared docs panel.
- User app upload widget + own docs panel.
- Text extraction action per `text-extraction-by-mime.md` (per-mime extractor, OCR fallback).
- Language detection on extracted text → `docs.lang`.
- Versioning + soft-delete + scheduled hard-purge per `doc-versioning-and-deletion-cascade.md`.
- Policy classifier on every upload per `policy-relevance-classifier.md`; admin policy editor (`/admin/policy`) + quarantine queue (`/admin/quarantine`) + user "Request review" button.
- `settings` table seeded with default `corpus_policy` text on first compose boot.

## P3 — Agent tools

- `docs list` CLI command + Convex action.
- `docs read` CLI + action.
- `docs grep` CLI + action (iterate matching rows, regex over content text).
- `docs diff` CLI + action.
- Tool registry codegen wired (`packages/cli/bin/x-codegen.ts`).
- Skill blob served at `/api/cli/skill` describes the CLI surface.

## P4 — Embedding + similar

- Ollama compose service running `nomic-embed-text-v2-moe`.
- Chunking action per `embedding-chunking-strategy.md`: sliding window 400/50 over `docs.extractedText`, writes `docChunks` rows.
- Embedding action POSTing to `http://ollama:11434/api/embed` with `search_document: ` prefix.
- Per-doc `docs.embedding` = centroid of `docChunks.embedding`.
- `vectorIndex` on both `docs.embedding` and `docChunks.embedding`.
- `docs similar` CLI + action (centroid by default; `--granular` for chunk-level).

## P5 — Chat UX

- Chat sidebar (thread list per user).
- Chat pane with streaming render via Convex reactive `streamEvents` subscription.
- Tool-call render (collapsible breadcrumbs).
- Citation chips linking back to doc viewer.
- Doc viewer (markdown preview, scrollable).

## P6 — Cost + audit

- Daily owner $ cap enforced in proxy (per `kimi-cost-rates-and-reservation.md`).
- Per-chat turn budget.
- Rate limits (owner + chat).
- Audit log query view (admin only).
- Audit retention cron per `audit-retention-and-purge-cron.md` (90d, configurable).
- Backup script + restore drill per `backups-pg-dump-and-restore-drill.md`.

## P7 — Verify + harden

- Pull real test corpus per `docs/adr/test-corpus-source-and-kimi-knowledge-probe.md`; Kimi-knowledge probe rejects docs Kimi already knows.
- End-to-end smoke covering: upload → scan → ingest → embed → chat → tool call → answer → citation.
- Cost-control proof: simulated burst exhausts budget, returns 402.
- Sandbox isolation proof: prompt-injection inside a doc trying to `curl` outside is blocked at network layer.
- Cross-user isolation proof: user A's docs invisible to user B's sandbox.
- ALL items in `VERIFY.md` pass.

## P8 — Assessment tests

- `topics`, `testQuestions`, `testQuestionSuggestions`, `testAttempts`, `testAssignments`, `testPasses` active in Convex.
- Question generation pipeline triggered by `docs.policyStatus` → `approved` (async via `scheduler.runAfter`).
- 10 questions per approved doc; Vietnamese-only; dup + contradiction scans.
- Admin review queue at `/admin/test-questions/pending` with per-card actions + bulk-approve.
- Soft cap 50 per topic with 1-in-1-out cap-swap pairs; admin stretch allowed.
- Pool < 5 blocks attempts + assignment creation.
- Attempt flow: random 5, 100% pass, shuffled order, pinned snapshot, open-book, reveal-on-pass.
- Assignment flow: admin to role=user, real-time fire, persistent per-assignment badge, un-assign nuke.
- Re-arm cascade on substantive-flagged batch commits.
- `training` provider exposed to chat agent (read-only own-data).
- `lastSubstantiveUpdate` field active on `topics`.

## P9 — Departments + dashboard + agent auto-assign

- `userProfiles` table active; admin sets `department` per user.
- `costRecords` table active; proxy settles per-(user, model, day) on every call.
- `/admin/dashboard` as default landing route.
- Top strip tiles (active/total users, cost cycle, docs-in-corpus) via reactive sub.
- Cost cycle 5th-to-5th: monthly chart + per-(user, model) pivot table + drill-ins.
- Gradebook matrix with `✓ ✗ ⓐ ·` glyphs + row totals + column footers + drill-ins.
- Department column on gradebook (no scope-gating in v0).
- Agent auto-assign daily cron (03:00 UTC) gated by `settings.agent_auto_assign_enabled`.
- One aggregate `auditLogs` row per cron run.

## P10 — Polish

- Empty states, error toasts, loading skeletons.
- Doc-viewer keyboard nav.
- Chat history export.
- Backups: daily `pg_dump` cron + tested restore path.
