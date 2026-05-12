# clamav-stateless-scan-service

A self-host ClamAV-based scan service (one Docker container) exposes `POST /scan` taking file bytes → returns `{ok, type, sha256, virus?}`. Stateless: no file storage on the scan side. Called by the upload service before any blob lands in Convex `_storage`.

## Beats

- **No scan**: every uploaded doc is potentially malicious. Internal team isn't a zero-threat environment.
- **Scan inline in upload handler**: pulls ClamAV into the upload server's process; harder to scale; harder to swap the scanner.
- **External scan SaaS (VirusTotal etc.)**: outbound network dest; violates self-host invariant.

## Real cost

- One more container in the compose stack.
- ClamAV signatures update daily (cron job inside container); old signatures = missed threats.
- ClamAV cold start ~30s on first scan (loads signature DB into memory). Subsequent scans warm.

## Gotcha for Claude

- The scan service is purely a function (bytes → verdict). State (the file in flight) lives in the upload service's staging dir, not the scan service.
- `INSTREAM` ClamAV command streams the file from socket; scan service exposes HTTP wrapper.
- Archive recursion limit (`MaxRecursion`) set explicitly; default 16 layers is generous, drop to 4 to defend against zip bombs.
- File size cap enforced before invoking scan (oversize → 413 at upload service, never reach scan).
- Magic-byte sniff via libmagic in the scan service (`file -b --mime-type`); don't trust `Content-Type` from the client.
- On scan pass: upload service moves staging file → Convex `_storage`; writes the `docs` row with `scanStatus='clean'`, `sha256=<from scan>`.
- On scan fail: upload service deletes staging file; writes a `docs` row with `scanStatus='quarantined'` for audit, no blob persisted, surfaces reason in UI.
