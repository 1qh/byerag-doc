# VERIFY

End-state checklist. Every item must pass before the project counts as launched per `book/PHILOSOPHY.md`. Re-run after every meaningful change; record outcomes in `ledger.jsonl`.

## Auth

- [ ] Admin app: Google OAuth sign-in completes; session cookie set; role stamped `admin`.
- [ ] User app: same on user app; role stamped `user`.
- [ ] Cross-app cookies don't leak: signing into user app does not grant admin app access.
- [ ] Sign-out clears the cookie; next request 401.
- [ ] CLI device-flow login completes; PAT issuance works; revocation breaks the token immediately.

## Upload

- [ ] Admin uploads a clean PDF; row appears in `docs` with `scope='shared'`, `scanStatus='clean'`.
- [ ] User uploads a clean PDF; row appears with `scope='mine'`, `owner=<user>`, `scanStatus='clean'`.
- [ ] EICAR test virus rejected; staging file deleted; UI shows quarantine reason.
- [ ] Oversized file (>configured cap) rejected at the upload endpoint before reaching scan.
- [ ] Zip bomb rejected by ClamAV with recursion-limit error.

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
