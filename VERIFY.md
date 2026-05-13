# VERIFY

End-state checklist. Every item must pass before the project counts as launched per `book/PHILOSOPHY.md`. Re-run after every meaningful change; record outcomes in `ledger.jsonl`.

## Auth

- [ ] Admin app: Google OAuth sign-in completes; session cookie set; role stamped `admin`.
- [ ] User app: same on user app; role stamped `user`.
- [ ] Cross-app cookies don't leak: signing into user app does not grant admin app access.
- [ ] Sign-out clears the cookie; next request 401.
- [ ] CLI device-flow login completes; PAT issuance works; revocation breaks the token immediately.

## Upload

- [ ] Admin uploads a clean PDF; row appears in `docs` with `scope='shared'`, `scanStatus='clean'`, `version=1`.
- [ ] User uploads a clean PDF; row appears with `scope='mine'`, `owner=<user>`, `scanStatus='clean'`, `version=1`.
- [ ] EICAR test virus → `docs` row with `scanStatus='quarantined'`, no blob; UI toast `⚠️ Your file was rejected because it appeared suspicious. Reason: <sig>.`; audit row recorded.
- [ ] Oversized file (>configured cap) rejected at the upload endpoint before reaching scan.
- [ ] Zip bomb rejected by ClamAV with recursion-limit error; same quarantine path.
- [ ] **Duplicate content** (re-upload same sha256, same scope): no new row; toast `this file is already in your library (uploaded as <filename> on <date>).`
- [ ] **Version conflict** (same filename, same scope, different content): blocking modal `a different file with this name already exists. Replace it? Keep both? Cancel?`
  - [ ] **Replace** → new row `version=2`, `supersedes=<prev>`; prev row gets `supersededBy=<new>` + `deletedAt=now`; prev blob scheduled for 30-day hard-purge.
  - [ ] **Keep both** → new row with filename suffix `(2)`; both rows independent (`supersedes=null`).
  - [ ] **Cancel** → no row, no blob; staged tmp file deleted.
- [ ] User app surfaces same dedup + version-conflict UX scoped to `mine`.
- [ ] Cross-scope dedup: shared doc with content X does NOT block a user uploading content X to `mine` (and vice versa).
- [ ] Repeated quarantine uploads of the same sha256 from the same uploader within 1 hour → 429 `too many rejected uploads`.

## Embedding

- [ ] Ollama running with `nomic-embed-text:v2-moe` model pulled.
- [ ] On clean upload: embedding written to `docs.embedding` within N seconds.
- [ ] `docs similar` query returns docs with cosine-rank ordering, filter pushdown by scope+owner verified.
- [ ] Matryoshka 256 / 512 / 768 dim queries all return results.

## Chat

- [ ] User starts a new chat in user app; row inserted with `app='user'`.
- [ ] User sends "list my docs"; agent calls `byerag docs list --scope mine`; result streams back.
- [ ] User asks a semantic question; agent calls `byerag docs similar` then `byerag docs read`; cites in answer.
- [ ] Admin app analog works on shared docs.
- [ ] Resume: close tab, reopen, prior chat history visible; resume on prior thread continues the SDK session.

## Sandbox

- [ ] First chat message spawns a Docker container with gVisor runtime (verified via `docker inspect`).
- [ ] Second message reuses the same container.
- [ ] `docker network ls` shows sandbox attached to restricted bridge, not default.
- [ ] From inside sandbox: `curl https://api.kimi.com/` fails (no DNS / no route).
- [ ] From inside sandbox: `curl <convex-internal>/api/anthropic/v1/messages` with valid bearer succeeds.
- [ ] On chat delete: container killed; `sandboxes` row removed.

## Proxy

- [ ] Valid bearer → LLM call forwards → response streams back.
- [ ] Invalid bearer → 401.
- [ ] Path other than `/v1/messages` → 403.
- [ ] Body over cap → 413.
- [ ] Burst exceeding per-chat rate → 429.
- [ ] Daily $ cap exhausted → 402.
- [ ] Per-chat turn budget exhausted → 429.
- [ ] Cost settled post-call: `ownerSpend.centsToday` reflects actual usage.

## Isolation

- [ ] User A uploads private doc. User B signed into user app cannot see it via `docs list --scope mine`.
- [ ] User B's sandbox `/workspace/mine/` mount contains B's docs only.
- [ ] User A's chat sandbox cannot reach user B's chat data via Convex (different per-chat secret).
- [ ] Prompt injection in a doc trying to `curl evil.com` fails at sandbox network layer.
- [ ] Prompt injection in a doc trying to read `/etc/passwd` fails at gVisor isolation.

## Audit

- [ ] Every CLI exec call recorded in `auditLogs`.
- [ ] Admin can query the log; user cannot.
- [ ] Sign-in events recorded.
- [ ] Log retention policy active (configurable; default 90 days).

## Operations

- [ ] `pg_dump` of Convex's backing Postgres completes; restore on a clean instance produces identical state.
- [ ] Host firewall blocks all egress except `api.kimi.com:443` (verified via `iptables -L` or equivalent).
- [ ] Convex backend restart preserves all data (volume mount validated).
- [ ] `bunx pm4ai@latest fix` exits silently on the byerag code repo.

## Performance baseline

- [ ] First-message latency end-to-end < 5s on a small corpus (≤100 docs).
- [ ] Tool-call round trip (Convex action via CLI) < 200ms on warm sandbox.
- [ ] Convex reactive streamEvent push to client < 100ms median.
- [ ] Embedding throughput: 1 doc / second on CPU baseline (acceptable for internal team).

## Doc lifecycle

- [ ] Same filename re-upload bumps `docs.version`; `supersedes` chain links walk-able.
- [ ] Soft-delete (set `deletedAt`) removes doc from `docs list` and `docs similar` results.
- [ ] Scheduled hard-purge after 30 days deletes `_storage` blob + `docChunks` rows; `docs` row retained for audit.
- [ ] Citation to a soft-deleted doc renders with "deleted" badge; doesn't 404.
- [ ] Citation to a superseded doc renders with version + "updated on" badge.

## Concurrency

- [ ] User opens two tabs of the same chat — second tab renders read-only banner.
- [ ] Heartbeat from active tab maintains token; >15s silence allows other tab to claim.
- [ ] "Take over" button on banner claims the token; original tab flips to banner.
- [ ] `messages.send` with mismatched `activeContextToken` returns 403.

## Doc extraction

- [ ] PDF with text layer: `extractedText` populated; OCR not triggered.
- [ ] Scanned PDF: OCR fallback runs; `extractedText` populated.
- [ ] docx / pptx / xlsx / epub: extraction works.
- [ ] Image (png/jpg/tiff): OCR extracts text.
- [ ] Unsupported mime: upload rejected at the upload endpoint.

## Embedding + chunking

- [ ] Doc with extractedText > 2K chars: chunks created in `docChunks` with sliding window 400/50.
- [ ] `docs.embedding` is centroid of `docChunks.embedding`.
- [ ] `docs similar --query X` returns docs ranked by cosine on centroid.
- [ ] `docs similar --query X --granular` returns (docId, chunkSeq, snippet, score).
- [ ] Filter pushdown: `--scope mine` excludes other users' chunks.

## CI + lint

- [ ] `bun run fix` exits silently on a clean tree.
- [ ] `bun run check:schema-drift` fails when `SCHEMAS.md` ≠ `schema.ts`.
- [ ] `bun run check:doc-leak` fails when a banned string is added to code.
- [ ] `bun run check:secret-leak` fails when a sk-/JWT-shaped string is added to a tracked file.
- [ ] CI workflow runs all of the above + sandbox image build smoke.

## Backups

- [ ] `apps/backend/scripts/backup.sh` produces an `age`-encrypted dump.
- [ ] `restore-drill.sh` recovers a parallel stack from latest dump; row counts match ±5%.
- [ ] Backup target disk is separate from the Postgres data disk.

## Network bridge

- [ ] From sandbox: `curl https://api.kimi.com/` fails (DNS not resolvable).
- [ ] From sandbox: `curl http://convex-backend:3210/api/anthropic/v1/messages` succeeds with valid bearer.
- [ ] From sandbox: `curl http://convex-backend:3210/api/anthropic/v1/messages` with INVALID bearer returns 401.
- [ ] From sandbox: `curl http://attacker.local/` (anything other than convex) fails at the iptables FORWARD rule.
- [ ] Host: `nft list ruleset` shows `output policy drop` with only Kimi IPs in the allowlist set.
