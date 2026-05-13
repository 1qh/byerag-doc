# dashboard-gradebook

Per-(user, topic) matrix on admin dashboard. Rows = role=user users, columns = non-deleted topics, cells = glyph per state. Row totals + column footer pass-rates.

## Cell glyphs

- `✓` — user passed this topic (any kind: assigned or self).
- `✗` — admin-source assignment exists, user not yet passed (any attempt state).
- `ⓐ` — agent-source assignment exists, user not yet passed.
- `·` — no assignment, not passed.

Conflict resolution: if both admin and agent assignments exist for the same `(user, topic)` (admin assigned first, agent later, or vice versa), render `✗` — admin takes visual priority.

## Layout

```
User                Dept    Topic A  Topic B  Topic C  Topic D  Total
─────────────────────────────────────────────────────────────────────
carol@x.com         HR      ✗        ⓐ       ·        ·        0/4
dan@x.com           Sales   ·        ✗        ✓        ⓐ        1/4
...
─────────────────────────────────────────────────────────────────
Topic pass rate             80%      40%      75%      33%
```

- **Dept** column: from `userProfiles.department` (null shown as "Unassigned").
- **Total**: `passed_count / assigned_count` per user (assigned = any non-deleted assignment OR existing pass).
- **Topic pass rate** footer: `passed_assigned / assigned` per topic (per `assessment-test-overview.md` invariant).

## Default sort

- Rows: by Total ratio ascending (most-struggling first). Tie-break: by lowest absolute passed count, then alphabetical.
- Columns: by `topics.createdAt` ascending (oldest first).

## Click drill-ins

- **Cell** → user-topic detail page: attempt history, source-doc citations from passed snapshots.
- **Row total** → user detail page: per-topic status + attempt history.
- **Column header (topic name)** → topic detail page: per-user status across all assigned users, question pool size, last substantive update.

## Refresh cadence

On-demand button. Aggregate query is heavy (rows × cols). Re-render only on explicit admin refresh, NOT live reactive sub.

## Dept filter

Department column shown but no v0 filter dropdown. Admin manually sorts the column visually. v1: dropdown to filter rows by `HR | Sales | IT | Unassigned`.

## Cell color cues

- `✓` green (success)
- `✗` red (attention)
- `ⓐ` blue (agent-suggested)
- `·` neutral grey

Color is decoration; glyph carries semantics for color-blind-safe.

## Beats

- **Per-cell live reactive sub** — too many cells × users × topics; bandwidth waste.
- **Tree-shaped (collapsed by dept)** — adds UI complexity; flat matrix is admin's mental model.
- **List view (one row per assignment)** — loses the cross-topic comparison.

## Real cost

- One aggregate Convex query per admin refresh.
- On large teams (>100 users) + many topics (>30), table scrolls heavily.

## Gotcha for Claude

- Admin-source and agent-source `testAssignments` rows can coexist for same `(user, topic)` — agent-assigned first, admin assigned later, or vice versa. Render priority = admin wins (`✗`).
- Passed-then-re-armed cells: after substantive update, prior `✓` flips to `✗` (re-arm inserts admin-source) or `ⓐ` (next cron). Glyph reflects post-re-arm state.
- Deleted topics: column hidden. Past pass records survive but invisible on gradebook (visible in user-detail drill-in).
- Pool < 5 topics: column hidden from gradebook (users can't take the test → meaningless column).
- Empty cells (`·`) include "agent eligible but hasn't run yet" — once cron runs, `·` flips to `ⓐ`. Manual cron trigger (deferred) would let admin force-fire.
- "Topics" count in gradebook header should equal non-deleted topics with pool ≥ 5.
