# training-page

`/admin/training` is the admin's single surface for assessment training: assignment health at a glance, per-topic assignment actions, and a searchable roster. Dashboard is stats-only and does NOT host any assignment surface.

## Deadlines

Assigned tests have a due date. Global admin setting `settings.assignment_due_days` (default `14`) is the standard window: `dueAt = createdAt + assignment_due_days × 86_400_000`, computed, not stored — changing it re-derives every overdue state instantly. The manual Assign composer can set a **per-batch override** stored as `testAssignments.dueAtMs`; effective due = the row's `dueAtMs` if present, else the global window. The agent never sets an override (stays global).

- **Overdue** = a `testAssignments` row, source `assigned` (admin or agent), `deletedAt=null`, the user has NOT passed it (no `testPasses` with `kind='assigned'`), and `now > effectiveDue`.
- Self-assessments (`testPasses.kind='self'`, self-started attempts) have no due date and never go overdue.
- Deadlines are admin-visibility only. Users are never shown a due date and are never blocked by it — open-book, unlimited-retake, no-cooldown posture is unchanged.

## Layout

Three KPI cards, a Tests table, a User-summary table, then the Assignments table. No persistent agent strip — agent activity surfaces as a transient toast.

### KPI cards

All three cards are clickable drill-downs (cards are single scannable numbers; the *who/what* lives in the tables below):

- **Overview** — total role=user accounts · % users who passed *all* assigned · overall assigned pass-rate. Click → scrolls to the User-summary table (no filter).
- **People at risk** — single count of users with ≥1 assigned test not yet passed. Click → Assignments table, Status filter = "Unfinished", scrolls to it.
- **Weakest test** — topic with the lowest assigned pass-rate (ties → largest assigned). Click → Assignments table filtered (search = that test name), scrolls to it.

### Agent notification (toast only)

No persistent agent strip. When a new **agent-sourced** assignment appears (the query's `latest` advances and `source==='agent'`), a transient toast fires: "Agent assigned <test> to <user>". First load sets the baseline silently (no toast). The on/off toggle for `agent_auto_assign_enabled` lives in the header controls; `settings.agent_last_check` still updates every enabled tick but is no longer surfaced inline.

### Tests table

One row per non-deleted topic with pool ≥ 5: name · questions (pool) · assigned · pass-rate · overdue. **Search by test name**; client-side paginated (10/page; control only shows when >1 page — expected scale under 20). Per-row `⋯` is topic-maintenance only — **Un-assign all** (`trainingAssignments.unassignAllForTopic`) and **Mark substantive (re-arm)** (`training.markTopicSubstantive`). Assigning is NOT here; it goes through the Assign composer. A topic *is* a test (its approved pool is the test) — one selector.

### User-summary table

Per-user aggregate (re-added alongside the per-assignment table — both kept; summary answers "is this person done?", Assignments answers "exactly which test/when"). Columns: **User · Department · Passed / assigned · Overdue**. Search by user, server-paginated 25, sorted worst-first (overdue, then unfinished). `api.dashboard.userSummary({page,search})`. The Overview card scrolls here.

### Assignments table

The per-assignment surface — one row per live `testAssignments` record, each row reads as a sentence. Columns: **User · Department · Test · Status · Deadline · Assigned** (dates in Vietnam time). Status plain words: `✓ Passed` · `⏰ Overdue N days` · `Not passed`. **Deadline** = effective due date (`dueAtMs` if set else `createdAt + global window`); overdue rows highlight the deadline. Sorted overdue → not-passed → passed, then most-recent.

Every filterable column header (**Department · Test · Status · Deadline · Assigned**) is a clean ghost-styled dropdown (the header *is* the control — no boxed inputs, no separate filter row) opening a **multi-select checkbox list** of the distinct values present (server-computed facets), with a count badge + Clear. User column is plain. Multi-select within a column = OR; across columns = AND. Server-paginated, 10/page. People-at-risk deep-links Status∈{Overdue, Not passed}; Weakest-test deep-links Test=that test.

Backed by `api.dashboard.assignmentsTable({page, departments[], tests[], statuses[], deadlines[], assigneds[]})` (admin-gated) → `{ rows[], latest, facets, pageCount, total }`. `facets` = distinct sorted values for each column (drives the dropdowns). Each row `{ userId, department, test, status, overdueDays, source, deadline, assigned, at }` — `deadline`/`assigned` are preformatted Vietnam-time date strings. `latest` drives the agent toast (when `source==='agent'`). Self-passes never appear. Admins excluded.

### Assign composer

Header **"Assign a test"** button opens a modal (`training.assignComposer`, admin-gated):

- **Test** — pick a topic (pool ≥ 5).
- **Audience** — Everyone · A department (Safety, Health and Environment, or Unassigned).
- **Overdue after (days)** — optional per-batch override → stored `testAssignments.dueAtMs`; blank = standard global window.
- Skips users who already passed (assigned) or have a live assignment. Audit `command='training.assign.runNow'`, `mode='admin'`, `owner='agent'`, `severity='medium'`.

The header has exactly two buttons: **Assign a test** (this composer) and **Agent auto-assign** (opens the agent dialog). The agent dialog carries: enable on/off, **Overdue after (days)**, **Assign at** (Vietnam-time hour select; "Continuously" = every tick, or a specific hour), **weekdays** multi-select (shown only when an hour is chosen; empty = every day), and an **Assign eligible now** button. Per `agent-auto-assign-cron.md`.

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
