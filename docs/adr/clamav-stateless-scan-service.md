# clamav-via-convex-action

Upload virus-scan is a Convex `'use node'` action talking directly to `clamd`'s TCP socket via `node:net`. ClamAV daemon runs as one compose service; no scan wrapper service in between.

## Mechanism

`internal.docs.scan({bytes}) → {ok, mime, sha256} | {ok: false, signature}`:

1. Connect TCP to `clamav:3310`.
2. Send `zINSTREAM\0`, 4-byte big-endian length, then bytes, then 4-byte zero terminator.
3. Read response; parse for `FOUND` / `OK`.
4. Compute sha256 + magic-byte mime sniff in parallel.

Action runs inside Convex's Node runtime. Upload mutation calls it inline; on pass writes the blob to `_storage` + the `docs` row.

## Beats

- **Separate scan microservice (Hono / Bun / Express wrapping clamd)**: extra container, extra hop, extra healthcheck. Pure middleman.
- **Scan in browser before upload**: trust-the-client breaks every threat model.
- **No scan**: every uploaded doc is potentially malicious.

## Real cost

- Convex action runtime hosts the TCP code path; one more reason an action is `'use node'`.
- File bytes live in action memory during the scan call. Upload bodies are capped to the doc-size limit anyway.

## Gotcha for Claude

- `zINSTREAM` (with leading `z`) returns NUL-terminated reply; without leading `z` returns newline-terminated. Pick one and parse consistently.
- 4-byte length is big-endian unsigned 32-bit. Encode with a DataView, not number-to-string.
- 4-byte zero terminator after the data block tells clamd "stream done".
- Archive recursion + size limits configured in `clamd.conf` (baked into the clamav image's mounted config). Adjust there, not in the action.
- Signature DB updates: clamav image runs `freshclam` on boot + periodically. First boot needs network access to fetch the initial DB (one-time exception; document in operator runbook).
- For very large files (>100 MB), streaming the blob from `_storage` through the action to clamd avoids loading the full bytes into action memory; use Bun's stream-to-socket pattern.
