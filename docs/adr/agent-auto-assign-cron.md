# agent-auto-assign-cron

Daily cron at fixed UTC time (default 03:00) walks every `(role=user user, eligible topic)` pair, inserts `testAssignments` with `createdBy='agent'` for empty `(·)` cells. Gated by admin's manual enable flag. No rate limit. No user-facing source distinction. Aggregate audit row per cron run.

## Trigger

Convex `crons.daily('agent-auto-assign', {hourUTC: 3, minuteUTC: 0}, internal.training.agentAutoAssign)`.

## Eligibility query

For each `(userId, topicId)` pair where:

- `userId` is a non-revoked auth account with `role='user'` (admins excluded).
- `topicId` is a non-deleted `topics` row with approved-question pool size ≥ 5.
- No `testPasses` row exists for `(userId, topicId, kind='assigned')`.
- No non-deleted `testAssignments` row exists for `(userId, topicId)` (regardless of source).

Insert `testAssignments` with:

- `userId`, `topicId`, `createdAt=now()`.
- `createdBy='agent'`.
- `deletedAt=null`.

## Gate

`settings` row with `key='agent_auto_assign_enabled'`, `value='false'` (default seeded on first compose boot). Admin flips to `true` via admin UI once after deploy. Cron checks flag at start; no-ops if false.

## No rate limit

Every eligible `(user, topic)` pair fills on every cron run. First cron after enable can produce hundreds of assignment rows in one tick.

## No admin notification

Cron run produces no notification. Admin sees results via gradebook `ⓐ` glyphs on next dashboard load.

## Admin-triggered immediate sweep

The daily 03:00 UTC cron is the standing automation; the enable flag gates ONLY that cron. Separately, the dashboard gradebook carries an **"Assign eligible now"** button that runs the **same deterministic eligibility sweep on demand**, so an admin does not wait until 03:00 UTC.

- Same logic as the cron: eligible topics (pool ≥ 5, not deleted) × role=user accounts → insert `testAssignments` for empty cells, skipping existing passes/live assignments. No LLM/agent inference is involved anywhere in auto-assign — "agent" is the product framing; rows are `createdBy='agent'` and surface as the `ⓐ` glyph so the user sees it as the agent's work.
- **Flag-independent.** Pressing the button IS the admin's explicit choice; it runs regardless of `agent_auto_assign_enabled`. The flag still gates only the cron. Enabling the flag does NOT itself trigger an immediate sweep — the button is the immediate path, the toggle is the recurring path.
- Admin-gated (caller must be role=admin). Distinct audit row: `command='training.assign.runNow'`, `mode='admin'`, `owner='agent'` (agent-attributed), `args` carries `triggeredBy=<admin email>` + counts. The cron keeps its own `training.cron.run` row — the two are never conflated.
- Idempotent: re-pressing only fills still-empty eligible cells (same skip rules as the cron).

## No admin override

- Admin's "un-assign topic" cascade nukes all rows for topic (admin + agent). Next cron refills if eligibility unchanged.
- No per-topic agent-lock toggle in v0. Admin who wants to suppress agent for a topic must delete the topic entirely.

## User-side transparency

User sees badge regardless of source. No "from agent" label. Source visible to admin only via gradebook glyph.

## Audit

One `auditLogs` row per cron run:

- `command='training.cron.run'`
- `args={topicsProcessed: N, assignmentsCreated: M, durationMs: T}`
- `mode='system'`
- `owner='agent'`
- `severity='low'`

Per-assignment audit rows NOT written (would spam log; aggregate per run sufficient).

## Beats

- **Continuous reactive trigger** — harder to reason about; cron timing is predictable.
- **Manual-only trigger** — admin operational burden; cron is set-and-forget.
- **On-corpus-update only** — misses time-based signals (re-test intervals, new users joining); cron sweeps all.

## Real cost

- One Convex action per day. Cheap.
- Burst of badge appearances on first cron after enable: locked tradeoff, admin briefs team.

## Gotcha for Claude

- Settings flag is the only kill switch in v0. Flip false → cron no-ops; existing `ⓐ` rows remain until admin un-assigns or topic deletes. Flip true → next cron refills.
- Concurrent admin assignment + cron run: idempotency holds. Cron checks for existing `testAssignments` row before insert; admin-source row inserted seconds before cron is seen → cron skips.
- Cron failure (timeout, transient DB error): retry on next day's cron. No mid-day retry.
- First cron after enable can produce hundreds of inserts in one transaction. Convex action time budget applies; if exceeded, partial completion is acceptable — next day's cron picks up the remainder.
- Cron audit row `severity='low'`; doesn't surface as high-severity alert. If admin wants tighter monitoring, query `auditLogs.by_command='training.cron.run'`.
- Substantive re-arm cascade (per `re-arm-on-substantive-corpus-update.md`) inserts admin-source assignments. Agent's cron sees those and skips. Re-arm is admin-source path, agent doesn't duplicate.
