# costrecords-table

Per-`(owner, model, dayKey)` aggregation table. Source of truth for cost analytics on the dashboard. Distinct from `ownerSpend` (single-row-per-day reservation/cap tracking).

## Schema

`costRecords`:

- `owner: string` — lowercase email
- `model: string` — e.g. `kimi-for-coding`
- `dayKey: string` — `YYYY-MM-DD` UTC
- `inputTokens: number`
- `cacheCreationInputTokens: number`
- `cacheReadInputTokens: number`
- `outputTokens: number`
- `cents: number`
- `callCount: number`

Indexes: `by_owner_model_dayKey`, `by_dayKey`, `by_owner_dayKey`.

## Write path

Proxy settle (post-LLM-response, per `proxy-per-chat-bearer-cost-controls.md`) increments the right row atomically:

```
upsert costRecords(owner, model, dayKey={today UTC}):
  inputTokens += usage.input_tokens
  cacheCreationInputTokens += usage.cache_creation_input_tokens
  cacheReadInputTokens += usage.cache_read_input_tokens
  outputTokens += usage.output_tokens
  cents += <computed actual cents per kimi-cost-rates-and-reservation.md>
  callCount += 1
```

**Non-chat Kimi spend is also recorded.** Question generation (`trainingGen`), the policy classifier (`docsPolicy`), and the `docs conflict` tool call Kimi directly (not via the chat proxy). Each parses the response `usage` and calls `internal.costRecords.recordDirect`, which computes cents with the same rate logic (`computeActualCents`) and upserts. Owner attribution: background pipelines (generation, classification) record under owner `'system'`; `docs conflict` is user-driven so it records under the calling user's email. This makes dashboard cost reflect ALL Kimi usage, not only chat-proxy traffic. Recording is best-effort and never blocks the operation.

`ownerSpend` continues to track per-day reservation/cap separately (different concern: live cap math vs analytics). Generation cost remains NOT budget-gated per `question-generation-pipeline.md` — recording it for visibility does not gate it.

## Read path

Dashboard cycle pivot: sum rows where `dayKey ∈ [cycleStart, cycleEnd)`, group by `(owner, model)`.

Dashboard chart: sum cents per cycle (group by cycle window).

Per-user drill-in: scan `by_owner_dayKey` for one user across the selected cycle's days.

## Beats

- **Per-call audit log only** — slow to aggregate; rows expire at 90d audit retention.
- **Reuse `ownerSpend`** — model dimension absent; can't pivot by model.
- **Read from `auditLogs`** — auditLogs purges at 90d; cost history needs longer.

## Real cost

- One row per `(owner, model, day)` per active user. Tiny at internal-team scale.
- Aggregate sums on every dashboard load for cycle windows.

## Retention

`costRecords` retained forever. Storage trivial (one row per user-model-day; ~1KB max per row). Cost history is operationally valuable indefinitely.

## Gotcha for Claude

- `dayKey` is UTC string; cycle math snaps to UTC midnight boundaries.
- `cents` is integer (cents, not USD); avoid floating point. Convert to dollars at display time.
- `callCount` increments per upstream call, including cache hits.
- If Kimi adds a new model name mid-deploy, new rows auto-fan-out by `model` field; no schema change.
- On proxy 5xx / partial failure with no usage, no row update. Cost lost. Acceptable: operator-side rare.
