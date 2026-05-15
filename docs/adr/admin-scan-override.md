# admin-scan-override

ClamAV-rejected uploads in the admin app trigger a standard yes/no confirm modal. Admin clicks `Yes` to force the file through, `No` to discard. User app has no such surface — virus = hard reject.

## Flow

Admin uploads → ClamAV reports `FOUND`. Server writes `docs` row with `scanStatus='quarantined'` and `storageId` set (blob persists in Convex `_storage`, isolated from search by the `scanStatus` gate — no row in `docs list` until override flips status). Override is a role-gated mutation keyed by `docId`; admin role + idempotency on `scanStatus==='quarantined'` is the safety surface (no rotating token).

UI renders a standard yes/no confirm modal:

```
⚠ Suspicious file detected.
Filename: <name>
Reason: <ClamAV signature>

Force upload?

[ No ]   [ Yes ]
```

`No` is default focus; Enter key does NOT trigger override.

`Yes` click → mutation `docs.adminScanOverride({docId})`:

1. Verify caller role `admin`.
2. Verify `docs.scanStatus === 'quarantined'` (idempotency — second click on flipped row no-ops).
3. Set `scanStatus='clean'`, `scanOverriddenBy=<admin email>`, `scanOverriddenAt=now()`, `scanOverrideSignature=<matched signature>`. Blob already in `_storage` from initial quarantine insert.
4. Proceed to next gate (policy classifier). Policy classifier may still reject the override'd file.
5. Audit log row `severity='high'`.

`No` click → `docs.adminScanCancel({docId})`: blob deleted from `_storage`, `storageId` nulled, `scanCancelledAt=now()` set (row retained for audit, no further state changes).

## User app

No override surface. User-app virus uploads = hard reject toast + audit row. No re-prompt, no override token, no `Request review` button for SCAN failures (policy failures only). Malware is not a policy disagreement; user-side review-request would invite social-engineering against the admin queue.

## Misclick acceptance

Team explicitly accepts yes/no misclick risk. No typed-confirmation friction; no delay-before-Yes-enabled. Audit `severity='high'` makes accidental overrides loudly visible after the fact. `No` as default focus is the only safety measure.

## Schema additions (`docs` table)

- `scanOverriddenBy: string?` — admin email
- `scanOverriddenAt: number?`
- `scanOverrideSignature: string?` — original ClamAV signature (audit)
- `scanCancelledAt: number?` — admin pressed `No`

Index: `by_scanOverriddenBy` (admin self-audit page).

## Audit log

Override → `auditLogs` row:

- `command: 'docs.scanOverride'`
- `args: JSON({docId, filename, sha256, signature})`
- `mode: 'admin'`
- `ok: true`
- `severity: 'high'`

## Beats

- **No override at all (hard reject)**: false positives block real work; admin has no recourse for a legitimate file ClamAV got wrong.
- **Typed-OVERRIDE confirmation**: rejected by team as too much friction for normal admin workflow. Yes/No standard pattern is more familiar; audit log catches accidents.
- **2-sec delay-before-Yes-clickable**: same friction problem; rejected.
- **Per-signature allowlist**: configuration burden + admin must understand ClamAV signature names.
- **Same override surface on user app**: invites abuse; lowers attack cost (every user becomes a potential admin-queue exploit).

## Real cost

- Two states on doc row (`scanStatus` + `scanOverriddenBy` independent).
- Staging blob retained 1h — small disk cost.
- One easy-to-misclick surface in admin app; audit + retrospective alerting compensates.

## Gotcha for Claude

- Mutation is idempotent on `scanStatus`; second click after override no-ops because `scanStatus !== 'quarantined'`. Admin closes tab + reopens → quarantine row visible in `/admin/quarantine` (scanStatus filter); admin can override/cancel from there.
- Re-uploaded file (same sha256) after previous override is NOT auto-cleared — ClamAV signature DB may have been updated; always re-scan, always re-prompt on fail.
- Policy classifier runs AFTER the override; an overridden virus that's also off-topic still gets rejected on the policy gate (admin can override that too via `/admin/quarantine`).
- Audit alerting (P6+) fires on `severity='high'` rows so override surfaces in real-time monitoring.
- `scanOverrideSignature` preserves original detection even after `scanStatus` flips to `clean` — required for incident response.
- Yes/No modal MUST default focus on `No`. Reduces accidental Enter-key activation.
