# audit-retention-and-purge-cron

`auditLogs` and `streamEvents` retained for 90 days. A scheduled function (`crons.purgeOldAudit`) runs daily, deletes rows older than the threshold.

## Mechanism

`apps/backend/convex/crons.ts`:

```ts
crons.daily('purge-old-audit', { hourUTC: 4, minuteUTC: 0 }, internal.audit.purge)
```

`internal.audit.purge`:

- Walks `auditLogs` by `_creationTime` ascending, batched 1000 rows per page.
- Deletes rows where `_creationTime < (now - 90d)`.
- Walks `streamEvents` by `_creationTime` ascending; deletes rows where `_creationTime < (now - 90d)` AND the parent `chat.streaming === false` (don't truncate live chats).

## Configurable

`AUDIT_RETENTION_DAYS` env (default 90). Bump for compliance windows; drop for storage pressure.

## Beats

- **Forever-retention**: storage grows linearly; eventual ops pain.
- **Per-row TTL**: Convex doesn't natively support TTL columns; cron is the pattern.
- **Hard-delete on no schedule**: ad-hoc deletion drifts.

## Real cost

- One daily action; ~O(deleted rows) work.
- Loses long-tail forensic data past 90 days.

## Gotcha for Claude

- `streamEvents` for active chats must NOT be purged mid-stream; the cron skips rows whose chat is still `streaming === true`.
- `messages` table is NOT purged — chat history is the user's record; only the streaming-event buffer is short-lived.
- Cron failure → `auditLogs` row records the failure; alerting (P6+) catches recurrence.
- Bump retention requires schema-untouched env change + `bun sync`; takes effect on next cron tick.
