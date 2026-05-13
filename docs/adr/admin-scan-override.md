# admin-scan-override

ClamAV-rejected uploads in the admin app trigger an irreversible-action confirmation. Admin may force the file through if they're certain it's a false positive. User app has no such surface — virus = hard reject.

## Flow

Admin uploads → ClamAV reports `FOUND`. Server writes `docs` row with `scanStatus='quarantined'`, blob retained in tmp staging dir for 1 hour pending decision (NOT yet moved to `_storage`). Server returns rejection response with a one-time override token (random UUID, 1-hour TTL).

UI renders a blocking confirmation modal:

```
⚠️ Suspicious file detected

Filename: <name>
Reason: <ClamAV signature>
SHA256: <hash>

This file matched a known malware or suspicious-content signature. Uploading it could expose the team to risk.

Type "OVERRIDE" to force this upload anyway, or click Cancel.

[OVERRIDE input]   [Cancel]   [Force upload]
```

The `Force upload` button is disabled until the input matches `OVERRIDE` exactly. Click → mutation `internal.docs.scanOverride({docId, token, typedConfirmation})` runs:

1. Verify caller's role is `admin`.
2. Verify token matches + is unexpired.
3. Verify `typedConfirmation === 'OVERRIDE'`.
4. Verify `docs.scanStatus === 'quarantined'` (idempotency).
5. Move staging blob to `_storage`; set `storageId`, `scanStatus='clean'`, `scanOverriddenBy=<admin email>`, `scanOverriddenAt=now()`, `scanOverrideSignature=<the matched signature>`.
6. Proceed to next gate (policy classifier). Policy classifier may STILL reject the override'd file.
7. Audit log records the override w/ ALL fields + a high-severity flag.

Cancel → blob deleted from staging immediately; row updated with `scanCancelledAt=now()` (kept for audit, no further state changes).

If the 1-hour TTL expires without decision, a scheduled function purges the staging blob; the row remains tombstoned.

## User app

No override surface. User-app uploads that fail scan get the standard rejection toast + audit row. No re-prompt, no override token, no `Request review` button for SCAN failures (only for policy failures). Rationale: malware is not a policy disagreement; allowing user-side review-request would invite social-engineering against the admin queue.

## Schema additions (`docs` table)

- `scanOverriddenBy: string?` — admin email
- `scanOverriddenAt: number?`
- `scanOverrideSignature: string?` — original ClamAV signature (for audit)
- `scanCancelledAt: number?` — admin pressed Cancel

Index: `by_scanOverriddenBy` (admin self-audit page).

## Audit log

Override creates an `auditLogs` row with:

- `command: 'docs.scanOverride'`
- `args: JSON({docId, filename, sha256, signature, typedConfirmation: '<redacted>'})`
- `mode: 'admin'`
- `ok: true`
- `severity: 'high'`

`auditLogs.severity` is a new optional field (`v.optional(v.union(v.literal('low'), v.literal('medium'), v.literal('high')))`) added to support filtering on critical events.

## Beats

- **No override at all (hard reject)**: false positives block real work; admin has no recourse for a legitimate file ClamAV got wrong.
- **One-click admin override (no typed confirmation)**: too easy to misclick; lowers the bar below what malware risk warrants.
- **Per-signature allowlist**: configuration burden + admin must understand ClamAV signature names.
- **Same override surface on user app**: invites abuse + lowers attack cost (every user becomes a potential admin-queue exploit).

## Real cost

- Two states on the doc row (`scanStatus` + `scanOverriddenBy` are independent fields).
- Staging blob retained 1h pending decision — small disk cost.
- Admin types one extra word; small friction with a very real safety payoff.

## Gotcha for Claude

- The typed confirmation must be **exact** match `OVERRIDE` — no case-insensitive, no trim, no partial. Easy to forget on the client; enforce on the server.
- Token is single-use; once consumed (override or cancel) it's deleted, second click is no-op. If admin closes the tab and reopens, they re-upload (no re-prompt for the abandoned staging — purge handles cleanup).
- A re-uploaded file (same sha256) after a previous override is NOT auto-cleared — ClamAV signature DB may have been updated, and the new scan might pass anyway or still fail. Always re-scan, always re-prompt on fail.
- Policy classifier runs AFTER the override; an overridden virus that's also off-topic still gets rejected on the policy gate (and admin can override that too via `/admin/quarantine`, but that's a separate override flow).
- Audit alerting (P6+) fires on `severity='high'` audit rows so the override surfaces in real-time monitoring, not just retrospective audit.
- `scanOverrideSignature` preserves the original detection even after the row's `scanStatus` flips to `clean` — required for incident response.
