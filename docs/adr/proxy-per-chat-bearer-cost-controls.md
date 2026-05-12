# proxy-per-chat-bearer-cost-controls

A Convex `httpAction` proxies all LLM traffic. Authentication is a per-chat bearer (`sk-ant-oat01-proxy_<chatId>_<secret>`). The real LLM key lives only in Convex env. Cost controls (daily owner cap, per-chat turn budget, rate limits) enforced at proxy.

## Beats

- **Direct LLM call from sandbox**: leaks the real key to the sandbox. Compromised sandbox = leaked key. No.
- **API gateway in front of Convex (Kong / Envoy)**: another service to run; cost controls split between gateway + Convex.
- **Per-owner bearer**: compromised sandbox in chat A could spend chat B's budget on user A. Per-chat scope tightens blast radius.

## Real cost

- Every LLM call adds a hop through Convex. Convex's `httpAction` is fast (<5ms overhead warm).
- Convex action body cap applies (8MB default; configurable).
- Streaming response passes through Convex; bandwidth doubles vs direct.

## Gotcha for Claude

- Per-chat secret stored as `chats.secretHash` (bcrypt-style hash, NEVER plaintext). Proxy constant-time-compares the bearer's secret against the hash.
- `ALLOWED_UPSTREAM_PATHS` allowlist gates the proxy: only `/v1/messages` + `/v1/messages/count_tokens` allowed. Everything else 403.
- `MAX_PROXY_BODY` caps request body; oversize → 413 before any forwarding.
- Cost reservation: pre-call estimate (worst-case input tokens × rate + max_tokens × output rate) reserved against `ownerSpend.centsToday`. Settled post-call with actual usage. Prevents concurrent over-spend.
- Per-chat turn budget (`chatRuntime.proxyCallsThisTurn`) decrements per call; exhausted → 429 with `proxy turn budget exhausted` message.
- Upstream URL re-validated after parse: `new URL(upstreamPath, 'https://api.kimi.com')` then `.host` checked == `api.kimi.com`. Defends against URL parse-confusion SSRF.
- Idle-stream abort: `AbortController` with 15s idle on the upstream fetch; prevents indefinite open connections from a misbehaving upstream.
