# REQUIREMENTS

Feature catalog. Canonical state, not transition. Each row is required for the launch-tier endpoint per `book/PHILOSOPHY.md`.

## Auth

- Google OAuth sign-in on both apps via `@convex-dev/auth`.
- Role stored on `userProfiles.role: 'admin' | 'user'` per `docs/adr/role-on-user-profile.md`.
- First sign-in: role defaults `'user'`. Bootstrap admin via env `BOOTSTRAP_ADMIN_EMAIL` (comma-separated allowed); first sign-in by matching email seeds `role='admin'`.
- Existing admins promote/demote other users via admin UI (impl deferred).
- Admin app `/admin/*` routes guarded by `userProfiles.role === 'admin'`. Non-admin → 403.
- User app: any signed-in account.
- Session persisted in Convex `authTables`.
- Sign-out clears session cookie.

## Upload pipeline (admin and user)

Every upload runs through these gates in order; failure at any gate ends the flow with the matching UI:

1. **Scan** (ClamAV) — malicious → `scanStatus='quarantined'`, blob retained in 1h staging.
   - **User app**: hard reject. Toast: `⚠️ Your file was rejected because it appeared suspicious. Reason: <signature>.` No appeal surface for scan failures.
   - **Admin app**: yes/no confirm modal `⚠ Suspicious file detected. Force upload?` with `[No] [Yes]` buttons. `Yes` → admin force-through per `admin-scan-override.md`; audit `severity='high'`. `No` → staging blob deleted.
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

- `docs list --scope shared|mine|both`
- `docs read --id <docId>`
- `docs grep --pattern <regex> --scope <s> [--limit N]`
- `docs diff --a <docIdA> --b <docIdB>` (mechanical unified diff)
- `docs conflict --a <docIdA> --b <docIdB>` (semantic conflict scan; LLM-driven; per `docs/adr/auto-resolve-via-shared-kb-on-conflict.md`)
- `docs similar --query <text> --scope <s> [--limit N] [--dim 256|512|768]`

When agent detects a factual conflict between two docs, system prompt instructs autonomous probe of shared-scope corpus for canonical authority + incorporate into answer with citations.

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

- On doc upload: text extracted (`pdftotext`, `pandoc`, etc.) → first `min(8K, content)` chars sent to local Ollama at `/api/embed` w/ `model="nomic-embed-text-v2-moe"`, prefix `search_document: `.
- Embedding stored as `docs.embedding` field, indexed via `vectorIndex` with `dimensions=768`, filter fields `owner`, `scope`.
- On `docs similar` tool call: query text prefixed `search_query: `, embedded, `ctx.vectorSearch` with ACL filter pushdown.

## Assessment tests

Per `docs/adr/assessment-test-overview.md` + siblings. Canonical behavior summary:

- Agent generates 10 MCQ candidates per approved shared doc; 3 choices each, 1 correct, Vietnamese-only.
- Agent flat-clusters topics; admin can delete (cascade). No rename/merge/split/lock/manual-create in v0. No prerequisites.
- All AI question changes flow through admin review queue. Source-doc deletion cascades automatically without review.
- Review actions: Approve / Edit / Reject. Bulk-approve via checkboxes only.
- Conflict pairs (dup or contradiction) and cap-swap pairs render as joint cards; admin resolves both items together.
- Soft cap 50 per topic; admin can stretch with warning.
- Pool < 5 blocks user attempts AND admin assignment creation. Topics with 0 hidden from user app entirely.
- Attempts: 5 random questions, 100% to pass, unlimited retakes, no cooldown, no time limit, open-book (source-doc citation inline + doc viewer beside), question+choice order shuffled per attempt, tab-close discards, one row per (user, topic).
- Reveal on pass only (full breakdown); failed attempt shows ratio only.
- Admin assigns to all `role=user`; admins exempt. Real-time fire via Convex reactive sub. Per-assignment persistent badge until passed. Un-assign nukes silently; in-progress attempts cancelled.
- Self-pass ≠ assignment-pass; separate `testPasses` rows per kind.
- Substantive corpus updates re-arm assigned-passes for affected topics. Admin classifies substantive/cosmetic at batch commit. Self-passes never re-arm.
- Chat agent gets read-only own-data tools: `training status / attempts / topics / attempt-detail`. No pool leak. No admin-curation access.
- Question generation cost not budget-gated. Per-user chat budget cap stays.

## Departments

Per `docs/adr/departments.md`.

- Three departments: HR, Sales, IT. Static enum.
- `userProfiles.department: 'HR' | 'Sales' | 'IT' | null`. Single department per user. Null for admins or unset.
- Department is a dashboard-filter dimension; does NOT gate assignment scope. Locked rule: `Assign to all users` includes every role=user account regardless of department.
- Admin sets each user's department via admin UI (impl deferred).
- Future v1: department-scoped assignment selectors.

## Dashboard (admin app)

Per `docs/adr/dashboard-admin-landing.md` + siblings.

- Admin lands on `/admin/dashboard` after sign-in.
- **Top strip** (live, reactive sub):
  - `active / total users` — heartbeat-based active count; total = non-revoked accounts.
  - Cost cycle (current). Monthly bar chart + per-(user, model) pivot table below.
  - Docs in corpus — count of approved, non-deleted shared docs.
- **Cost analytics** (per `docs/adr/dashboard-cost-cycle.md`):
  - 5th-to-5th cycle. Anchor day fixed at 5; future configurable via `settings.billing_cycle_day`.
  - Monthly bar chart, scrollable history, current cycle bar partial + live.
  - Pivot table: per-`(user, model)` rows w/ input/output tokens + cost. Sort by cost desc. Footer totals.
  - Click bar → re-render pivot for that cycle. Click pivot row → user daily breakdown drill-in.
- **Training page** `/admin/training` (per `docs/adr/training-page.md`) — assignment surface; dashboard is stats-only:
  - KPI cards: Overview, Overdue tests, People at risk, Weakest test.
  - Deadlines: `settings.assignment_due_days` (default 14); overdue = assigned-source, not passed, past due. Admin-only lens; users never see/blocked.
  - Tests table: one row per non-deleted topic with pool ≥ 5 (name · pool · assigned · pass-rate · overdue) + per-topic ⋯ (Assign to all / selected / Un-assign / Mark substantive).
  - Header: agent auto-assign on/off + Assign eligible now. Agent capability callout + activity feed (recent agent assignments) + "last checked" heartbeat — visible proof the agent is working.
  - Users table: paginated, columns username/email · department · passed/assigned · overdue; search by username; row expands inline to that user's per-test status (plain language).
  - No glyph matrix.

## Agent auto-assign

Per `docs/adr/agent-auto-assign-cron.md`.

- 5-minute Convex ticker; when enabled, each tick fills any newly-eligible cell (idempotent, no schedule, no burst).
- Gated by `settings.agent_auto_assign_enabled` (default `false`). Admin flips the single on/off on the Training page.
- Eligibility: pool ≥ 5 AND user not passed (kind='assigned') AND no live assignment.
- Inserts `testAssignments` with `createdBy='agent'`.
- No rate limit; every eligible cell fills.
- No admin notification; admin sees updated assigned/overdue counts on the Training page.
- No per-topic agent-lock toggle; admin un-assign nukes all rows but next cron refills if conditions hold.
- User sees no source distinction in badges.
- One aggregate `auditLogs` row per cron run.

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
- Sandbox container attaches to a restricted Docker bridge (`sandbox-egress`) with iptables FORWARD rules allowing only the Convex container's IP+ports per `docs/adr/network-bridge-rules.md`. DNS isolated via baked `/etc/hosts`.
- Backups via daily `pg_dump` of Convex's backing Postgres.
