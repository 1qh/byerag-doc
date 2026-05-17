# agent-auto-assign-cron

When the admin enables it, the agent **continuously** keeps every eligible `(role=user user, topic)` cell assigned. A Convex ticker runs every 5 minutes; each tick fills only newly-eligible gaps. No admin schedule, no cron expression, no time math — flipping one switch is the entire interface. Gated by the enable flag. No rate limit. No user-facing source distinction. Aggregate audit row only when a tick actually creates assignments.

## Trigger

Convex `crons.interval('agent auto-assign scheduler', { minutes: 5 }, internal.training.autoAssign)`. Each tick: if the enable flag is off, no-op (no audit); otherwise run the eligibility sweep. The sweep is idempotent — it skips users already passed or already assigned — so steady state creates zero rows and there is no daily burst. New hires, newly-live topics, and accidental un-assigns are picked up within minutes.

## Eligibility query

For each `(userId, topicId)` pair where:

- `userId` is a non-revoked auth account with `role='user'` (admins excluded).
- `topicId` is a non-deleted `topics` row with approved-question pool size ≥ 5.
- No `testPasses` row exists for `(userId, topicId, kind='assigned')`.
- No non-deleted `testAssignments` row exists for `(userId, topicId)` (regardless of source).

Insert `testAssignments` with `userId`, `topicId`, `createdAt=now()`, `createdBy='agent'`, `deletedAt=null`.

## Gate

`settings` row `key='agent_auto_assign_enabled'`, `value='false'` (seeded on first compose boot). Admin flips to `true` on the Training page. There is no schedule setting — "agentic" means the admin does not configure timing; the agent just keeps things current while enabled.

## No admin notification

No push notification. Every enabled tick overwrites `settings.agent_last_check` (epoch ms) — this powers the Training page heartbeat ("Agent last checked <rel time>") so the admin sees the agent is alive even on zero-result ticks, without audit-log spam. Sweeps that create rows also appear in the Training page activity feed. Per `training-page.md`.

## Admin-triggered immediate sweep

The Training page header carries an **"Assign eligible now"** button running the **same deterministic sweep on demand**, regardless of the enable flag (the button is the admin's explicit choice). Distinct audit row: `command='training.assign.runNow'`, `mode='admin'`, `owner='agent'` (agent-attributed), `args` carries `triggeredBy=<admin email>` + counts. No LLM/agent inference is involved anywhere in auto-assign — "agent" is the product framing; rows are `createdBy='agent'`.

## No admin override

- Admin's "un-assign topic" cascade nukes all rows for the topic (admin + agent). Next tick refills if eligibility still holds.
- No per-topic agent-lock toggle in v0. To suppress the agent for a topic, delete the topic.

## User-side transparency

User sees the assignment badge regardless of source. No "from agent" label. Source is an internal `createdBy='agent'` attribution; admin sees only aggregate assigned/overdue counts.

## Audit

One `auditLogs` row only on a tick that creates ≥ 1 assignment: `command='training.cron.run'`, `args={topicsProcessed, assignmentsCreated, durationMs}`, `mode='system'`, `owner='agent'`, `severity='low'`. Ticks that create nothing (steady state, flag off) write NO row — at a 5-minute cadence that would be ~288 rows/day of noise.

## Beats

- **Admin-set cron expression** — rejected: a cron string is the opposite of agentic and unusable by a non-technical admin. The agent should just keep training current, not ask the admin to schedule it.
- **Daily 03:00 batch** — solved a burst problem the continuous design doesn't have (idempotent ticks trickle, never flood) and delayed new-hire assignment by up to a day.
- **Continuous reactive trigger (DB hooks)** — more moving parts than a 5-minute idempotent sweep; the sweep is predictable and trivially correct.

## Real cost

- One cheap Convex action every 5 minutes; steady state does almost nothing (all cells already filled).
- First sweep after enable can create many rows at once: locked tradeoff, admin briefs the team.

## Gotcha for Claude

- Enable flag is the only control. Flip false → ticks no-op; existing agent rows remain until admin un-assigns or topic deletes. Flip true → next tick refills.
- Idempotency holds under concurrent admin assignment: the sweep checks for an existing `testAssignments` row before insert; an admin-source row inserted seconds earlier is seen and skipped.
- First sweep after enable can produce many inserts; Convex action time budget applies — partial completion is fine, the next tick (5 min later) finishes the remainder.
- Substantive re-arm cascade (per `re-arm-on-substantive-corpus-update.md`) inserts admin-source assignments; the agent sees and skips them — no duplication.
