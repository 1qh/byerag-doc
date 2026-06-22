# dashboard-cost-cycle

Cost analytics anchored to a 5th-to-5th monthly cycle. Top number, monthly bar chart, per-(user, model) pivot table — all driven by `costRecords` aggregation over the selected cycle window.

## Cycle definition

- Cycle = `[<5th of month N>, <5th of month N+1>)`.
- Labeled by start-month name + year (e.g. "May 2026" = `2026-05-05` → `2026-06-05`).
- Anchor day fixed at 5 in v0; future config via `settings.billing_cycle_day`.

## Components

### Top number

`$X.XX since May 5` — sum of cents in current cycle / 100. Updates live as proxy settles new spend.

### Monthly bar chart

- One bar per cycle. Horizontal scroll for history.
- Default visible: last 6 cycles.
- Current cycle bar = partial; grows live with new spend.
- Past cycles = frozen.
- Click bar → pivot table re-renders for that cycle. Top number flips too.

### Pivot table (below chart)

Per-(user, model) rows in selected cycle:

```
User       Model              Input tokens  Output tokens  Cost
alice@x    kimi-for-coding    142,300       8,420         $24.10
bob@x      kimi-for-coding    98,200        5,100         $19.40
...
─────────────────────────────────────────────────────────────────
Total                         380,600       20,620        $70.50
```

- Sort: cost desc.
- Footer: totals.
- Click row → drill-in: that user's per-day breakdown chart for the cycle.

## Aggregation

Pivot query: `costRecords` where `dayKey ∈ [cycleStart, cycleEnd)`, group by `(owner, model)`, sum tokens + cents + callCount.

Chart query: same window per cycle, sum cents only, one number per cycle.

## Refresh cadence

- Top number + current cycle bar: live (reactive sub).
- Pivot table: live for current cycle; static for past cycles (cycle data frozen).
- History bars: static (only current cycle changes).

## Beats

- **Single-day window** — admin wants monthly compliance/finance cycle.
- **Calendar month (1st-to-1st)** — operator's billing cycle is 5th-anchored.
- **Aggregate $ only** — pivot per (user, model) catches runaway user/model.

## Real cost

- `costRecords` writes on every proxy settle (one row touched per call).
- Cycle window can span 31+ days; sum aggregates run on every dashboard load.

## MUST

- Iterate `costCycleHistory` by calendar month (previous month's 5th), never a fixed `30 days × i` step. Why: a 30-day step accumulates error vs the 5th-of-month anchor, snapping two iterations into one cycle so a month renders twice or is skipped.
- Carry the year in the bar label (`MMM YYYY`), not the yearless `MM-DD` of `cycleStart.slice(5)`. Why: a `12-05` label crossing a year boundary reads as suspicious without the year.

## Gotcha for Claude

- 5th-day anchor: at midnight UTC of the 5th, current cycle flips. Implementation must snap cycle boundaries to UTC, not local time.
- Current cycle bar partial-ness: admin should see it visually distinct (e.g. striped, different shade) to distinguish from completed cycles.
- Pivot's "Model" column in v0 always shows `kimi-for-coding` (single LLM). Reserved for future multi-model deployments.
- Pivot rows include only users with non-zero spend in the window; users with $0 are not listed.
- Cache token columns (`cache_creation_input_tokens`, `cache_read_input_tokens`) absorbed into the `Input tokens` column unless admin clicks drill-in for the full detail.
