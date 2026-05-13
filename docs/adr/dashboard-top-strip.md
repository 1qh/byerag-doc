# dashboard-top-strip

Top of `/admin/dashboard`. Three live tiles. Convex reactive subscription on each.

## Tiles

### Active / total users

Format: `<active> / <total>` (e.g. `8 / 47`).

- **Active**: count of `userProfiles.userId` having `userContexts.activeContextHeartbeatAt > now() - 30s`. Live via reactive sub.
- **Total**: count of auth-table users not revoked/deleted (active accounts, role-agnostic). Stable; only changes on user mgmt.

### Cost cycle

See `dashboard-cost-cycle.md`. Top number = $ in current cycle (5th-to-5th). Monthly bar chart + pivot table below.

### Docs in corpus

Format: integer count.

Query: `docs` where `policyStatus='approved' AND deletedAt is null AND scope='shared' AND scanStatus='clean'`.

Cumulative; never resets.

## Refresh cadence

All three tiles use Convex reactive subscription. Updates push to admin's browser within seconds of underlying change.

## Beats

- **Snapshot-with-refresh-button** — admin doesn't want to click to see "is anything happening right now?"
- **Polled-every-N-sec** — Convex reactive sub is the cheaper natural primitive.

## Real cost

- One reactive query per tile (3 total).
- Heartbeat-based active count: 5s heartbeat cadence (per `concurrency-and-active-context-token.md`).

## Gotcha for Claude

- "Active" = heartbeat within 30s. User w/ tab open but idle 60+ sec → not counted.
- "Total" includes admins. If you want role=user only, separate count.
- Tile click → drill-in pages (active users list, full cost detail, doc library).
