# training-page

`/admin/training` is the admin's single surface for assessment training: assignment health at a glance, per-topic assignment actions, and a searchable roster. Dashboard is stats-only and does NOT host any assignment surface.

## Deadlines

Assigned tests have a due date. Global admin setting `settings.assignment_due_days` (default `14`) is the standard window: `dueAt = createdAt + assignment_due_days × 86_400_000`, computed, not stored — changing it re-derives every overdue state instantly. The manual Assign composer can set a **per-batch override** stored as `testAssignments.dueAtMs`; effective due = the row's `dueAtMs` if present, else the global window. The agent never sets an override (stays global).

- **Overdue** = a `testAssignments` row, source `assigned` (admin or agent), `deletedAt=null`, the user has NOT passed it (no `testPasses` with `kind='assigned'`), and `now > effectiveDue`.
- Self-assessments (`testPasses.kind='self'`, self-started attempts) have no due date and never go overdue.
- Deadlines are admin-visibility only. Users are never shown a due date and are never blocked by it — open-book, unlimited-retake, no-cooldown posture is unchanged.

## Layout

Four KPI cards, then a Tests table, then a Users table.

### KPI cards

- **Overview** — total role=user accounts · % users who passed *all* their assigned tests · overall assigned pass-rate (`passed_assigned / assigned` across all topics) · total overdue (user, topic) pairs.
- **Overdue tests** — total overdue pairs + top 3 worst-offender users (most overdue first).
- **People at risk** — top 5 users ranked by count of assigned-but-not-passed tests (overdue weighted first, then plain unpassed). Worst first.
- **Weakest test** — topic with the lowest assigned pass-rate (ties broken by largest assigned count).

### Tests table

One row per non-deleted topic with pool ≥ 5: name · pool size · assigned count · pass-rate · overdue count. The per-row `⋯` menu is topic-maintenance only — **Un-assign all** (`trainingAssignments.unassignAllForTopic`) and **Mark substantive (re-arm)** (`training.markTopicSubstantive`, per `re-arm-on-substantive-corpus-update.md`). Assigning is NOT in the row menu; it goes through the Assign composer.

A topic *is* a test (its approved question pool is the test). "Which test" = "which topic" — one selector.

### Assign composer

Header **"Assign a test"** button opens a modal (`training.assignComposer`, admin-gated):

- **Test** — pick a topic (pool ≥ 5).
- **Audience** — Everyone · A department (HR/Sales/IT/Unassigned) · Selected users (the Users-table checkboxes).
- **Overdue after (days)** — optional per-batch override → stored `testAssignments.dueAtMs`; blank = standard global window.
- Skips users who already passed (assigned) or have a live assignment. Audit row `command='training.assign.runNow'`, `mode='admin'`, `owner='agent'`, `severity='medium'`, args carry topic name + audience + counts.

Header also carries agent automation: **Agent auto-assign** on/off (`settings.agent_auto_assign_enabled`, per `agent-auto-assign-cron.md`) and **Assign eligible now** (`training.assignEligibleNow`, deterministic immediate sweep, agent-attributed). No schedule UI — agentic means the admin does not configure timing.

## Agent visibility panel

Directly below the header, an always-visible panel makes the agent's work provable, not mysterious:

Default state is a **single collapsed status strip** (one row, no scroll cost) directly under the header so the KPI cards remain the first real content:

- Strip leads with the **latest agent action** as the at-a-glance proof — `· last assigned N tests <rel time>`. Falls back to `· running · everyone eligible already assigned (checked <rel time>)` when there is nothing to do, or `· assignments are manual only` when off. No "starting up…" inert state — a non-tech admin reads that as broken.
- Bot icon (lucide `Bot`), tinted by state — no emoji.
- **Details ▾** expands in place: the capability list (assigns all eligible, catches new hires, assigns newly-live tests, refills accidental un-assigns) + an **activity table** — paginated (10/page), **searchable by test name**, columns When · Source (Agent/Manual) · Assigned · Tests. Backed by enriched `auditLogs` (`training.cron.run` + `training.assign.runNow` rows carry a `topics` name list, capped 30).
- Heartbeat data is `settings.agent_last_check` (overwritten every enabled tick; not an audit row, to avoid log spam).

`api.dashboard.agentActivity({page,search})` (admin-gated) returns `{ events[], lastCheck, pageCount, total }`; `events[]` carry `topics: string[]`.

### Users table

Paginated roster of role=user accounts: username/email · department (null → "Unassigned") · passed/assigned ratio · overdue count. Per-row select checkbox feeds "Assign to selected". **Search by username/email** (substring, case-insensitive). Pagination server-side, page size 25.

Clicking the username **expands the row inline** to a plain-language list of that user's assigned tests, one line each: `✓ <test> — passed`, `● <test> — not passed`, or `⏰ <test> — overdue (N days)`. One row open at a time. This replaces the matrix's per-cell status with a per-person readable view; no glyph grid, no navigation away.

Admins are excluded from the roster (admin curates content; testing them is theater, per `assessment-test-overview.md`).

## Refresh cadence

Card + table queries are on the standard reactive subscription (cheap aggregates over indexed scans at internal-team scale). No manual "Load" button — the page is live on open.

## Beats

- **Per-(user,topic) glyph matrix** — dense, cryptic for a non-technical admin; replaced by the roster + per-topic Tests table. Cross-topic-at-a-glance need is served by the KPI cards (weakest test, people at risk).
- **Per-assignment explicit due date** — admin friction every assign; the agent cron has no sensible date to pick. One global window is the trivially-maintainable knob; per-topic override can layer later if a real need appears.
- **User-facing deadlines / lockout** — breaks the open-book, unlimited-retake trust posture. Deadlines are an admin lens only.

## Real cost

- KPI + tables: a few indexed aggregate scans per page open; trivial at internal-team scale.
- Users table paginated server-side so a large roster does not ship every row.

## Gotcha for Claude

- Overdue is derived, not stored. No migration when `assignment_due_days` changes — recompute on read.
- Admin-source and agent-source `testAssignments` can coexist for the same `(user, topic)`. Overdue counts the pair once, not per source.
- Re-armed assignments (substantive update deletes the prior `testPasses.assigned`) get a fresh `createdAt` via the new assignment row, so the due clock restarts from re-arm time.
- Pool < 5 topics hidden from the Tests table; pre-existing assignments on a since-shrunk topic still count toward overdue until passed or un-assigned.
- Self-passes never satisfy assignment rows and never count as overdue; skip/overdue logic keys on `kind='assigned'` only.
- "Tests" row count equals non-deleted topics with pool ≥ 5.
