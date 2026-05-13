# timestamps-and-timezone

All timestamps in DB rows and `ledger.jsonl` are UTC milliseconds since epoch (Convex's default `number` for time). All ISO-8601 strings include `Z` suffix. UI renders in the browser's local time.

## DB

`number` (epoch ms). Stored as `v.number()` in schema. Convex sorts on these natively. Avoid `Date` objects across the wire.

## Ledger

`ledger.jsonl` row's `at` field is ISO-8601 with `Z`: `"2026-05-13T14:23:55Z"`. Programmatic queries use `Date.parse`; humans read directly.

## Audit logs

`auditLogs` rows carry `_creationTime` (Convex-provided, epoch ms) implicitly. No additional time field.

## UI rendering

Render via `Intl.DateTimeFormat` in the browser's locale + timezone. Show relative ("2 minutes ago") for recent events; absolute UTC + local for older.

## Beats

- **DB-side timezone-aware timestamps**: portability cost; Convex doesn't natively support it.
- **Strings everywhere**: parsing cost on every read.
- **Local time in DB**: cross-timezone admins see ambiguous timestamps.

## Real cost

- Mild discipline cost to convert at boundaries.

## Gotcha for Claude

- Daily cap `dayKey` is `YYYY-MM-DD` UTC, not local. Reset happens at midnight UTC, not user's midnight. Acknowledge in admin UI.
- Cron schedules expressed in UTC (`0 3 * * *` = 3 AM UTC daily backup).
- Time comparisons in code: `Date.now()` is epoch ms UTC; no conversion needed.
- Stale-checks (heartbeat, sandbox lastUsedAt) use `Date.now() - row.field > THRESHOLD_MS`. Always epoch math.
