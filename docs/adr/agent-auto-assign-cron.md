# agent-auto-assign-cron

When the admin enables it, the agent assigns every eligible `(role=user user, topic)` cell automatically. A Convex ticker runs every 5 minutes. The admin sets *when* via the **Agent auto-assign** dialog: either **continuously** (every tick — `agent_auto_assign_hour=''`) or on a **schedule** (a specific Vietnam-time hour, optionally restricted to chosen weekdays). The sweep is idempotent (skips already-passed / already-assigned), so continuous mode never bursts. Gated by the enable flag. No rate limit. No user-facing source distinction. Audit row only when a sweep actually creates assignments.

## Trigger

Convex `crons.interval('agent auto-assign scheduler', { minutes: 5 }, internal.training.autoAssign)`. Each tick: if `agent_auto_assign_enabled!=='true'` → no-op. Else read `agent_auto_assign_hour` / `agent_auto_assign_weekdays`:

- `hour===''` → continuous: run the sweep every tick.
- `hour` set → run only when the current Vietnam-time hour equals it, the VN weekday is in `agent_auto_assign_weekdays` (empty = every day), and `agent_auto_assign_last_run` is not already in the current VN hour bucket (one run per scheduled hour).

On an actual run it sets `agent_auto_assign_last_run=now`. Vietnam time = UTC+7, no DST.

## Eligibility query

For each `(userId, topicId)` pair where:

- `userId` is a non-revoked auth account with `role='user'` (admins excluded).
- `topicId` is a non-deleted `topics` row with approved-question pool size ≥ 5.
- No `testPasses` row exists for `(userId, topicId, kind='assigned')`.
- No non-deleted `testAssignments` row exists for `(userId, topicId)` (regardless of source).

Insert `testAssignments` with `userId`, `topicId`, `createdAt=now()`, `createdBy='agent'`, `deletedAt=null`.

## Gate

`settings` row `key='agent_auto_assign_enabled'`, `value='false'` (default). Admin flips it + sets the schedule in the **Agent auto-assign** dialog (one of the two header buttons on the Training page; the other is the admin **Assign a test** composer). The dialog also carries the overdue-after-days setting and an **Assign eligible now** button.

## No admin notification

No push notification. A new agent-sourced assignment surfaces as a transient toast on the Training page (per `training-page.md`); sweeps that create rows also write the `training.cron.run` audit row. No heartbeat surface.

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
