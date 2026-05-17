# training-page

`/admin/training` is the admin's single surface for assessment training: assignment health at a glance, per-topic assignment actions, and a searchable roster. Dashboard is stats-only and does NOT host any assignment surface.

## Deadlines

Assigned tests have a due date. Global admin setting `settings.assignment_due_days` (default `14`) is the standard window: `dueAt = createdAt + assignment_due_days × 86_400_000`, computed, not stored — changing it re-derives every overdue state instantly. The manual Assign composer can set a **per-batch override** stored as `testAssignments.dueAtMs`; effective due = the row's `dueAtMs` if present, else the global window. The agent never sets an override (stays global).

- **Overdue** = a `testAssignments` row, source `assigned` (admin or agent), `deletedAt=null`, the user has NOT passed it (no `testPasses` with `kind='assigned'`), and `now > effectiveDue`.
- Self-assessments (`testPasses.kind='self'`, self-started attempts) have no due date and never go overdue.
- Deadlines are admin-visibility only. Users are never shown a due date and are never blocked by it — open-book, unlimited-retake, no-cooldown posture is unchanged.

## Layout

Three KPI cards, a one-line agent status strip, a Tests table, then the Assignments table.

### KPI cards

- **Overview** — total role=user accounts · % users who passed *all* their assigned tests · overall assigned pass-rate (`passed_assigned / assigned`).
- **People at risk** — single count of users with ≥1 assigned test not yet passed. The whole card is a button → sets the Assignments-table Status filter to "Unfinished" and scrolls to it.
- **Weakest test** — topic with the lowest assigned pass-rate (ties broken by largest assigned count).

Principle: KPI cards are single scannable numbers. Drill-down is the Assignments table, never duplicated on a card.

### Agent status strip

One line under the header (no expand, no Details): lucide `Bot` icon tinted by state · "Agent on/off" · the latest assignment as proof (`· last assigned <test> to <user> <rel time>`), falling back to `· running · everyone eligible already assigned (checked <rel time>)` or `· assignments are manual only`. Heartbeat data is `settings.agent_last_check` (overwritten every enabled tick; not an audit row).

### Tests table

One row per non-deleted topic with pool ≥ 5: name · questions (pool) · assigned · pass-rate · overdue. **Search by test name**; client-side paginated (10/page; control only shows when >1 page — expected scale under 20). Per-row `⋯` is topic-maintenance only — **Un-assign all** (`trainingAssignments.unassignAllForTopic`) and **Mark substantive (re-arm)** (`training.markTopicSubstantive`). Assigning is NOT here; it goes through the Assign composer. A topic *is* a test (its approved pool is the test) — one selector.

### Assignments table

The single per-assignment surface — one row per live `testAssignments` record, the shape a non-tech admin acts on (each row reads as a sentence). Columns: **User · Department · Test · Status · Assigned** (exact datetime, Vietnam time). Status is plain words: `✓ Passed` · `⏰ Overdue N days` · `Not passed`. Sorted overdue → not-passed → passed, then most-recent.

Filters: **search** (user or test, substring) · **Department** dropdown (HR/Sales/IT/Unassigned/All) · **Status** dropdown (All / Unfinished / Overdue / Not passed in time / Passed). Server-paginated, 25/page. The People-at-risk card deep-links here with Status=Unfinished.

Backed by `api.dashboard.assignmentsTable({page,search,department,status})` (admin-gated) → `{ rows[], latest, lastCheck, pageCount, total }`; each row `{ userId, department, test, status, overdueDays, at }`. `latest` (+`lastCheck`) feed the status strip. Effective due per row = `dueAtMs` if set else global window. Self-passes never appear (only `testAssignments`). Admins excluded (not role=user).

### Assign composer

Header **"Assign a test"** button opens a modal (`training.assignComposer`, admin-gated):

- **Test** — pick a topic (pool ≥ 5).
- **Audience** — Everyone · A department (HR/Sales/IT/Unassigned).
- **Overdue after (days)** — optional per-batch override → stored `testAssignments.dueAtMs`; blank = standard global window.
- Skips users who already passed (assigned) or have a live assignment. Audit `command='training.assign.runNow'`, `mode='admin'`, `owner='agent'`, `severity='medium'`.

Header also carries **Agent auto-assign** on/off (`settings.agent_auto_assign_enabled`, per `agent-auto-assign-cron.md`) and **Assign eligible now** (`training.assignEligibleNow`). No schedule UI — agentic means the admin does not configure timing.

## Refresh cadence

Card + table queries are on the standard reactive subscription (cheap aggregates over indexed scans at internal-team scale). No manual "Load" button — the page is live on open.

## Beats

- **Per-(user,topic) glyph matrix** — dense, cryptic for a non-technical admin; replaced by the flat Assignments table (one plain-language row per (user, test): User · Department · Test · Status · Assigned). Same data, readable, filterable — not a grid of symbols.
- **Per-assignment explicit due date** — admin friction every assign; the agent cron has no sensible date to pick. One global window is the trivially-maintainable knob; per-topic override can layer later if a real need appears.
- **User-facing deadlines / lockout** — breaks the open-book, unlimited-retake trust posture. Deadlines are an admin lens only.

## Real cost

- KPI + tables: a few indexed aggregate scans per page open; trivial at internal-team scale.
- Assignments table paginated server-side so a large (users × tests) set does not ship every row.

## Gotcha for Claude

- Overdue is derived, not stored. No migration when `assignment_due_days` changes — recompute on read.
- Admin-source and agent-source `testAssignments` can coexist for the same `(user, topic)`. Overdue counts the pair once, not per source.
- Re-armed assignments (substantive update deletes the prior `testPasses.assigned`) get a fresh `createdAt` via the new assignment row, so the due clock restarts from re-arm time.
- Pool < 5 topics hidden from the Tests table; pre-existing assignments on a since-shrunk topic still count toward overdue until passed or un-assigned.
- Self-passes never satisfy assignment rows and never count as overdue; skip/overdue logic keys on `kind='assigned'` only.
- "Tests" row count equals non-deleted topics with pool ≥ 5.
