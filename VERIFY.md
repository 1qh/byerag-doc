# VERIFY

End-state checklist. Every item must pass before the project counts as launched per `book/PHILOSOPHY.md`. Re-run after every meaningful change; record outcomes in `ledger.jsonl`.

## Auth

- [ ] Google OAuth sign-in completes on either app; session cookie set.
- [x] First sign-in creates `userProfiles` row with `role='user'` by default.
- [x] Bootstrap: email matching `BOOTSTRAP_ADMIN_EMAIL` env (comma-separated) gets `role='admin'` seeded on first sign-in.
- [x] Admin app `/admin/*` routes: signed-in user with `role='admin'` → access granted; non-admin → 403.
- [x] User app routes: any signed-in user grants access.
- [x] Promote: admin changes another user's `role` to `'admin'`; after that user's next session refresh, admin routes are accessible.
- [x] Demote: admin changes a user's `role` to `'user'`; admin routes return 403 after session refresh.
- [x] Self-demotion prevented if it would leave zero admins; UI blocks.
- [ ] Sign-out clears the cookie; next request 401.
- [ ] CLI device-flow login completes; PAT issuance works; revocation breaks the token immediately.

## Upload

- [x] Admin uploads a clean PDF; row appears in `docs` with `scope='shared'`, `scanStatus='clean'`, `version=1`.
- [x] User uploads a clean PDF; row appears with `scope='mine'`, `owner=<user>`, `scanStatus='clean'`, `version=1`.
- [x] EICAR test virus → `docs` row with `scanStatus='quarantined'`, no blob; UI toast `⚠️ Your file was rejected because it appeared suspicious. Reason: <sig>.`; audit row recorded.
- [x] Oversized file (>configured cap) rejected at the upload endpoint before reaching scan.
- [ ] Zip bomb rejected by ClamAV with recursion-limit error; same quarantine path.
- [x] **Duplicate content** (re-upload same sha256, same scope): no new row; toast `this file is already in your library (uploaded as <filename> on <date>).`
- [x] **Version conflict** (same filename, same scope, different content): blocking modal `a different file with this name already exists. Replace it? Keep both? Cancel?`
  - [x] **Replace** → new row `version=2`, `supersedes=<prev>`; prev row gets `supersededBy=<new>` + `deletedAt=now`; prev blob scheduled for 30-day hard-purge.
  - [x] **Keep both** → new row with filename suffix `(2)`; both rows independent (`supersedes=null`).
  - [ ] **Cancel** → no row, no blob; staged tmp file deleted.
- [ ] User app surfaces same dedup + version-conflict UX scoped to `mine`.
- [x] Cross-scope dedup: shared doc with content X does NOT block a user uploading content X to `mine` (and vice versa).
- [x] Repeated quarantine uploads of the same sha256 from the same uploader within 1 hour → 429 `too many rejected uploads`.
- [ ] **Admin scan override** (admin app only):
  - [ ] Admin uploads a virus file → yes/no confirm modal `⚠ Suspicious file detected. Force upload?` with `[No] [Yes]` buttons.
  - [ ] `No` is default focus; Enter key does NOT trigger override.
  - [x] `Yes` click → server verifies role=admin + token + idempotency; on pass, blob moves to `_storage`, `scanStatus='clean'`, `scanOverriddenBy` + `scanOverriddenAt` + `scanOverrideSignature` populated.
  - [x] `Yes` click → audit log row with `severity='high'`, `command='docs.scanOverride'`.
  - [x] `No` click → staging blob deleted; row keeps `scanStatus='quarantined'` + `scanCancelledAt` set.
  - [x] 1-hour TTL expires without decision → scheduled function purges staging blob; row tombstoned.
  - [x] User app: no override surface visible; user with virus file gets hard reject toast only.
  - [x] Override does NOT bypass the policy classifier — virus-overridden doc still goes through policy gate; can still be rejected there.

## Policy classifier

- [x] Clean, on-topic doc → `policyStatus='approved'`, classifier `category='on-topic'`, doc searchable.
- [x] Off-topic doc (e.g. a novel chapter) → `policyStatus='rejected'`, `category='off-topic'`, UI toast `This file is rejected as not matching our policy. Reason: <reason>.` + `Request review` button visible.
- [x] Prompt-injection doc (content tries to manipulate the assistant) → `policyStatus='rejected'`, `category='prompt-injection'`. Admin override of this category emits an extra audit warning.
- [x] Spam/promotional doc → `policyStatus='rejected'`, `category='spam'|'promotional'`.
- [x] Abusive content → `policyStatus='rejected'`, `category='abusive'`.
- [x] `Request review` button → marks `policyReviewRequestedAt`; admin sees in queue.
- [x] Admin approves a rejected doc in `/admin/quarantine` → `policyStatus='approved'`, `policyOverriddenBy=<admin>`; doc becomes searchable; audit log records override.
- [x] Admin confirms reject in `/admin/quarantine` → blob purged immediately; row retained with `storageId=null`; audit log records.
- [x] Classifier failure (timeout / 5xx) → `policyStatus='pending'` retained; retry once after backoff; if both fail, surface to admin queue as "classifier error" with manual review.
- [x] Classifier cost (~$0.001/upload) deducted from uploader's `ownerSpend`; daily cap exhausted → upload blocked w/ standard 402 message.
- [x] Policy text editable by admin via `/admin/policy`; saves audited; new policy applies to subsequent uploads only.
- [x] Classifier output rendering in toast: HTML escaped, capped at 200 chars, no script-shaped patterns.
- [x] Request-review rate limit: 1 per file per uploader per day.

## Embedding

- [x] Ollama running with `nomic-embed-text-v2-moe` model pulled.
- [x] On clean upload: embedding written to `docs.embedding` within N seconds.
- [x] `docs similar` query returns docs with cosine-rank ordering, filter pushdown by scope+owner verified.
- [x] Matryoshka 256 / 512 / 768 dim queries all return results.

## Chat

- [x] User starts a new chat in user app; row inserted with `app='user'`.
- [x] User sends "list my docs"; agent calls `docs list --scope mine`; result streams back.
- [x] User asks a semantic question; agent calls `docs similar` then `docs read`; cites in answer.
- [x] Admin app analog works on shared docs.
- [x] Resume: close tab, reopen, prior chat history visible; resume on prior thread continues the SDK session.
- [x] **Supportiveness bar (per `AGENT-DOCTRINE.md` + evidence gate `docs/adr/supportiveness-evidence-gate.md`):** every check below validated via `apps/backend/scripts/smoke-supportiveness.ts` w/ scripted scenario + auto-judge + captured JSON in `apps/backend/test-fixtures/supportiveness-evidence/<scenario-id>.json`. Ledger notes reference the JSON path per tick.
  - [x] Cross-reference scenario: agent's answer mentions related shared-corpus obligation not in user's question. Captured `cross-reference-proactively.json` verdict=pass.
  - [x] Risk-spot scenario: agent flags auto-renewal clause unsolicited. Captured `spot-risks-unsolicited.json` verdict=pass.
  - [x] Dot-connect scenario: agent computes date arithmetic across offer letter + bonus policy. Captured `connect-dots-multi-doc.json` verdict=pass.
  - [x] Pre-empt scenario: agent covers carryover + parental leave intersection w/o being asked. Captured `pre-empt-follow-ups.json` verdict=pass.
  - [x] Gap-flag scenario: agent says "not in corpus" + recommends action. Captured `flag-corpus-gaps.json` verdict=pass.
  - [x] Uncertainty scenario: agent surfaces ambiguity instead of picking one interpretation. Captured `surface-uncertainty.json` verdict=pass.
  - [x] Citations + tool-call breadcrumbs: every factual claim has `<docId§section>` chip; stream contains ≥1 `tool_use` block. Captured `citations-and-breadcrumbs.json` verdict=pass.
- [x] **Conflict resolution flow** (real-world example per `docs/adr/auto-resolve-via-shared-kb-on-conflict.md`):
  - User uploads doc A (offer letter saying "15 days PTO") and doc B (PTO policy saying "20 days").
  - User asks "compare these 2 docs".
  - Agent autonomously runs `docs read` × 2 + `docs conflict --a --b` + `docs similar --scope shared` + `docs read` on canonical hit.
  - Final answer cites all three sources with `<docId§section>` chips.
  - When no canonical exists (no shared doc cosine ≥ 0.8), answer says "no shared-corpus authority found; recommend escalating to admin."
  - Excerpts in `docs conflict` output are literal substrings of source texts (server grep-verified; hallucinations dropped).
  - Hard cap 3 canonical probes per user-question respected.

## Sandbox

- [x] First chat message spawns a Docker container with gVisor runtime (verified via `docker inspect`).
- [x] Second message reuses the same container.
- [x] `docker network ls` shows sandbox attached to restricted bridge, not default.
- [ ] From inside sandbox: `curl https://api.kimi.com/` fails (no DNS / no route).
- [ ] From inside sandbox: `curl <convex-internal>/api/anthropic/v1/messages` with valid bearer succeeds.
- [ ] On chat delete: container killed; `sandboxes` row removed.

## Proxy

- [x] Valid bearer → LLM call forwards → response streams back.
- [x] Invalid bearer → 401.
- [x] Path other than `/v1/messages` → 403.
- [x] Body over cap → 413.
- [x] Burst exceeding per-chat rate → 429.
- [x] Daily $ cap exhausted → 402.
- [x] Per-chat turn budget exhausted → 429.
- [x] Cost settled post-call: `ownerSpend.centsToday` reflects actual usage.

## Isolation

- [x] User A uploads private doc. User B signed into user app cannot see it via `docs list --scope mine`.
- [ ] User B's sandbox `/workspace/mine/` mount contains B's docs only.
- [ ] User A's chat sandbox cannot reach user B's chat data via Convex (different per-chat secret).
- [x] Prompt injection in a doc trying to `curl evil.com` fails at sandbox network layer.
- [ ] Prompt injection in a doc trying to read `/etc/passwd` fails at gVisor isolation.

## Audit

- [x] Every CLI exec call recorded in `auditLogs`.
- [x] Admin can query the log; user cannot.
- [x] Sign-in events recorded.
- [x] Log retention policy active (configurable; default 90 days).

## Operations

- [ ] `pg_dump` of Convex's backing Postgres completes; restore on a clean instance produces identical state.
- [ ] Host firewall blocks all egress except `api.kimi.com:443` (verified via `iptables -L` or equivalent).
- [ ] Convex backend restart preserves all data (volume mount validated).
- [ ] `bunx pm4ai@latest fix` exits silently on the byerag code repo.

## Performance baseline

- [ ] First-message latency end-to-end < 5s on a small corpus (≤100 docs).
- [x] Tool-call round trip (Convex action via CLI) < 200ms on warm sandbox.
- [x] Convex reactive streamEvent push to client < 100ms median.
- [x] Embedding throughput: 1 doc / second on CPU baseline (acceptable for internal team).

## Doc lifecycle

- [x] Same filename re-upload bumps `docs.version`; `supersedes` chain links walk-able.
- [x] Soft-delete (set `deletedAt`) removes doc from `docs list` and `docs similar` results.
- [x] Scheduled hard-purge after 30 days deletes `_storage` blob + `docChunks` rows; `docs` row retained for audit.
- [x] Citation to a soft-deleted doc renders with "deleted" badge; doesn't 404.
- [x] Citation to a superseded doc renders with version + "updated on" badge.

## Concurrency

- [x] User opens two tabs of the same chat — second tab renders read-only banner.
- [x] Heartbeat from active tab maintains token; >15s silence allows other tab to claim.
- [x] "Take over" button on banner claims the token; original tab flips to banner.
- [x] `messages.send` with mismatched `activeContextToken` returns 403.

## Doc extraction

- [x] PDF with text layer: `extractedText` populated; OCR not triggered.
- [x] Scanned PDF: OCR fallback runs; `extractedText` populated.
- [x] docx / pptx / xlsx / epub: extraction works.
- [x] Image (png/jpg/tiff): OCR extracts text.
- [x] Unsupported mime: upload rejected at the upload endpoint.

## Embedding + chunking

- [x] Doc with extractedText > 2K chars: chunks created in `docChunks` with sliding window 400/50.
- [x] `docs.embedding` is centroid of `docChunks.embedding`.
- [x] `docs similar --query X` returns docs ranked by cosine on centroid.
- [x] `docs similar --query X --granular` returns (docId, chunkSeq, snippet, score).
- [x] Filter pushdown: `--scope mine` excludes other users' chunks.

## CI + lint

- [ ] `bun run fix` exits silently on a clean tree.
- [x] `bun run check:schema-drift` fails when `SCHEMAS.md` ≠ `schema.ts`.
- [x] `bun run check:doc-leak` fails when a banned string is added to code.
- [x] `bun run check:secret-leak` fails when a sk-/JWT-shaped string is added to a tracked file.
- [ ] CI workflow runs all of the above + sandbox image build smoke.

## Backups

- [x] `apps/backend/scripts/backup.sh` produces an `age`-encrypted dump.
- [x] `restore-drill.sh` recovers a parallel stack from latest dump; row counts match ±5%.
- [ ] Backup target disk is separate from the Postgres data disk.

## Test corpus

- [x] `apps/backend/scripts/pull-test-corpus.ts` runs successfully on first invocation.
- [x] For every candidate doc, Kimi-knowledge probe executes with no doc context.
- [ ] Docs Kimi knows (cosine sim > 0.85 OR exact-match facts) are rejected; probe-log records.
- [x] At least 5 real docs accepted (post-2026-02-01 sources).
- [ ] Edge-case fixtures present: EICAR string, prompt-injection doc, scan-only PDF, mixed VN+EN doc, oversized file, zip bomb.
- [x] Probe log file (`apps/backend/test-fixtures/probe-log.jsonl`) gitignored.
- [x] Pulled docs gitignored (operator-local fixtures).

## Assessment tests

### Generation pipeline

- [x] Admin uploads a shared doc → scan-clean + policy-approved → background gen fires; 10 candidates land in `testQuestionSuggestions` w/ `status='pending'`.
- [x] Doc gen async — doc shows in library + chat immediately; candidates lag ~10-30 sec.
- [x] Each candidate Vietnamese; technical terms preserved original.
- [x] Each candidate has `choices.length === 3`, `correctIndex ∈ {0,1,2}`.
- [x] Dup scan: candidates with cosine ≥ 0.85 vs existing pool flagged via `conflictsWith`.
- [x] Contradiction scan: paired retire-suggestion emitted with `pairKind='conflict'`.
- [x] At-cap (≥50 approved): new candidate gets paired retire with `pairKind='cap-swap'`.

### Admin review queue

- [x] Per-item actions: Approve / Edit / Reject / Regenerate (+ optional hint).
- [x] `regenCount` capped at 5; further regens disabled with message.
- [x] Bulk-approve via checkboxes works across multiple selected items.
- [ ] Conflict-pair card: Accept swap / Keep old / Keep both / Reject both atomic.
- [ ] Cap-swap card: same options; Keep both stretches pool with banner.
- [x] Approve writes canonical `testQuestions`; suggestion `status='resolved'`.
- [x] Source-doc delete → pending suggestions auto-rejected w/ `resolvedReason='source-doc-deleted'`.
- [x] Topic delete → pending suggestions auto-rejected w/ `resolvedReason='topic-deleted'`.

### Canonical admin actions

- [x] Admin can edit approved question (prompt/choices/correctIndex); `revision` increments; audit `severity='low'`.
- [x] Admin force-regenerate (+ hint) → new suggestion enters queue w/ `kind='revision'`; audit `severity='medium'`.
- [x] Admin retire → `deletedAt` set; audit `severity='medium'`.
- [x] No manual-from-scratch create UI exposed.

### Pool soft cap

- [x] Default cap 50 stored as `topics.poolCap`.
- [ ] 51st approval (stretch path) succeeds with banner.
- [x] Admin can adjust `poolCap` per topic.

### Topic management

- [x] Topics flat (no parentId on row).
- [x] Topic delete cascades: questions soft-delete w/ `deleteReason='topic-cascade'`; pending suggestions auto-rejected; assignments cancelled; in-progress attempts cancelled.
- [x] Empty topic (pool=0) hidden from user app's training page.
- [x] 0 < pool < 5: visible to user, `Start` disabled.
- [x] Pool ≥ 5: testable.

### Attempts

- [x] Start → server picks 5 random from approved pool; shuffles questions + choices; pins snapshot.
- [x] Question + choice order varies per attempt.
- [x] `kind='assigned'` if user has pending assignment; else `'self'`.
- [x] Pool < 5 → `Start` server returns 400.
- [x] Submit all correct → `status='passed'`; `testPasses` row inserted/updated for `(user, topic, kind)`.
- [x] Submit any wrong → `status='failed'`; no `testPasses` write.
- [x] Submit other user's attempt → 403.
- [x] `attempt-detail` on passed → full pinned snapshot w/ `correctIndexShuffled`.
- [x] On failed/cancelled → only `{score, total: 5}`.
- [x] Retake: new attempt for same (user, topic) atomically deletes prior row.
- [x] No time limit; no cooldown; no rate limit.
- [x] Tab close + restart → fresh random 5; orphan flips to `cancelled` on new insert.

### Assignments

- [x] `Assign to all` with pool < 5 → 400.
- [x] `Assign to all` with pool ≥ 5 → rows inserted for every role=user except those w/ existing `testPasses(kind='assigned')`.
- [x] Admins excluded from "all users".
- [ ] Real-time fire via Convex reactive sub.
- [ ] Offline user sees badge on next sign-in.
- [x] Badge persists until passing via `kind='assigned'` attempt.
- [x] Re-fire skips active passes; no duplicate rows.
- [x] Un-assign → all assignment rows `deletedAt` set; badges vanish via reactive sub; in-progress assigned-kind attempts → `cancelled`; past `testPasses` retained.
- [x] Un-assign audit `severity='medium'`.

### Substantive update re-arm

- [ ] Admin batch w/ at least one `'retire'` approval → default substantive; admin can flip cosmetic.
- [ ] Admin batch w/ only `'new'` approvals → default cosmetic; admin can flip substantive.
- [ ] Admin batch w/ only `'revision'` approvals → default substantive.
- [x] Source-doc deletion cascade → automatic substantive (no admin override).
- [x] Substantive commit writes `topics.lastSubstantiveUpdate=now()`.
- [x] Re-arm cascade: `testPasses` rows where `kind='assigned' AND passedAt < lastSubstantiveUpdate` deleted; fresh assignments inserted; audit `command='training.assignment.rearm'`.
- [x] Self-passes never re-armed.

### Chat agent training tools

- [x] `training status` returns caller-scoped topic status list.
- [x] `training attempts` returns caller-scoped attempt list.
- [x] `training topics` returns topic list w/ pool sizes; no question content.
- [x] `training attempt-detail --id X` on caller's passed attempt → full snapshot.
- [x] Same tool on caller's failed/cancelled attempt → only score.
- [x] Same tool on another user's attempt → 403.
- [ ] Agent refuses pool-content questions before user passes (prompt-injection-style "what's on the Security test?").

## Departments

- [x] `userProfiles` table active w/ `role ∈ {'admin', 'user'}` and `department ∈ {'HR', 'Sales', 'IT', null}`.
- [x] Admin sets a user's department via admin UI; audit row recorded.
- [x] Department NULL for admin role accounts.
- [x] Department NULL for unset role=user accounts (group "Unassigned" on gradebook).
- [x] `Assign to all` includes all role=user regardless of department.
- [x] Department visible on gradebook row.

## Dashboard

### Landing

- [x] Admin signs in → lands on `/admin/dashboard`, not docs library.

### Top strip

- [ ] `active / total` tile updates live via reactive sub when user heartbeat changes / accounts created/revoked.
- [x] Cost cycle tile shows current cycle's $ from 5th of month onward; flips at UTC midnight of next 5th.
- [x] Docs-in-corpus tile = count of `docs` where `policyStatus='approved' AND deletedAt=null AND scope='shared' AND scanStatus='clean'`. Updates live.

### Cost cycle

- [ ] Monthly bar chart renders bar per cycle; current cycle bar is partial + visually distinct.
- [ ] Past cycle bars frozen.
- [ ] Click past bar → top number + pivot table re-render for that cycle.
- [x] Pivot table: rows per `(user, model)` summed over selected cycle window.
- [x] Columns: User · Model · Input tokens · Output tokens · Cost.
- [x] Sort: cost desc.
- [ ] Footer row: totals.
- [ ] Click pivot row → user's daily breakdown chart for cycle.
- [x] In v0, every Model column shows `kimi-for-coding`.

### Gradebook

- [x] Rows: every non-revoked role=user account. Admins excluded.
- [x] Columns: every non-deleted `topics` row with pool ≥ 5. Pool < 5 topics hidden from gradebook.
- [x] Cells render correct glyph per state.
- [x] `✓` covers any pass (self or assigned).
- [x] `✗` admin-source live assignment + no pass.
- [x] `ⓐ` agent-source live assignment + no pass.
- [x] `·` no live assignment + no pass.
- [x] Concurrent admin + agent assignment renders `✗` (admin priority).
- [ ] Row total = `passed_count / assigned_count` per user.
- [ ] Column footer = `passed_assigned / assigned` per topic.
- [ ] Default sort: rows by Total asc, cols by `topics.createdAt` asc.
- [ ] Cell click → user-topic detail page.
- [ ] On-demand refresh button re-runs aggregate query.

## Agent auto-assign

- [x] `settings.agent_auto_assign_enabled` defaults `'false'` on first compose boot.
- [x] Cron at 03:00 UTC; no-op when flag is `'false'`.
- [x] Admin flips flag to `'true'` → next cron tick fires.
- [x] Per cron run, walks `(role=user, topic where pool ≥ 5 AND not deleted)`.
- [x] Skips `(user, topic)` w/ existing `testPasses(kind='assigned')`.
- [x] Skips `(user, topic)` w/ non-deleted `testAssignments`.
- [x] Inserts `testAssignments` w/ `createdBy='agent'` for eligible empty cells.
- [x] No rate limit per user; first cron after enable can produce hundreds of rows.
- [x] No admin notification.
- [x] No user source-label on badges (admin-source and agent-source look identical).
- [x] One aggregate `auditLogs` row per cron: `command='training.cron.run'`, `args={topicsProcessed, assignmentsCreated, durationMs}`, `mode='system'`, `owner='agent'`, `severity='low'`.
- [x] Admin un-assignment removes both admin + agent rows for topic; next cron refills eligible cells.
- [x] Cron failure: no mid-day retry; next day's cron picks up.

## Network bridge

- [x] From sandbox: `curl https://api.kimi.com/` fails (DNS not resolvable).
- [x] From sandbox: `curl http://convex-backend:3210/api/anthropic/v1/messages` succeeds with valid bearer.
- [x] From sandbox: `curl http://convex-backend:3210/api/anthropic/v1/messages` with INVALID bearer returns 401.
- [x] From sandbox: `curl http://attacker.local/` (anything other than convex) fails at the iptables FORWARD rule.
- [ ] Host: `nft list ruleset` shows `output policy drop` with only Kimi IPs in the allowlist set.
