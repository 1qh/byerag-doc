# ADR index

One file per decision. Format: pick + why-beat-alternatives + real-cost + non-obvious-gotcha.

## Foundation

- [`clone-and-strip-substrate-reference`](./clone-and-strip-substrate-reference.md) — bootstrap byerag from the substrate reference; strip product-domain layers
- [`pm4ai-monorepo`](./pm4ai-monorepo.md) — pm4ai-managed monorepo, lintmax single-fix command
- [`bun-only-toolchain`](./bun-only-toolchain.md) — bun is the only package manager + runtime
- [`nextjs-16-app-router`](./nextjs-16-app-router.md) — Next.js 16 RSC + Turbopack

## Backend + auth

- [`convex-self-host-on-this-machine`](./convex-self-host-on-this-machine.md) — Convex backend in local Docker compose
- [`google-oauth-via-convex-auth`](./google-oauth-via-convex-auth.md) — `@convex-dev/auth` Google provider
- [`role-by-app-not-allowlist`](./role-by-app-not-allowlist.md) — role determined by app the user signed into; no email allowlist

## Agent + LLM

- [`coding-agent-as-driver-not-rag`](./coding-agent-as-driver-not-rag.md) — agent drives tools; not a fixed retrieval pipeline
- [`kimi-as-llm`](./kimi-as-llm.md) — Kimi via Anthropic-protocol-compatible endpoint
- [`agent-sdk-inside-sandbox`](./agent-sdk-inside-sandbox.md) — Claude Agent SDK runs in the sandbox, not on the host
- [`proxy-per-chat-bearer-cost-controls`](./proxy-per-chat-bearer-cost-controls.md) — Convex httpAction proxy with per-chat bearer

## Sandbox + network

- [`docker-gvisor-sandbox`](./docker-gvisor-sandbox.md) — Docker + gVisor (runsc) per owner
- [`egress-only-llm-host`](./egress-only-llm-host.md) — only LLM API host reachable; sandbox network-off

## Apps + UX

- [`two-apps-admin-and-user`](./two-apps-admin-and-user.md) — separate Next.js apps for admin and user roles
- [`shared-react-package-for-chat-ui`](./shared-react-package-for-chat-ui.md) — `packages/react` carries chat plumbing for both apps
- [`streamevents-reactive-pipeline`](./streamevents-reactive-pipeline.md) — `streamEvents` table + Convex reactive sub feeds the UI

## Docs corpus

- [`docs-table-via-convex-storage`](./docs-table-via-convex-storage.md) — `docs` row + `_storage` blob; no external blob store
- [`clamav-stateless-scan-service`](./clamav-stateless-scan-service.md) — self-host ClamAV called as a function (bytes → verdict)
- [`ollama-nomic-embed-v2-moe`](./ollama-nomic-embed-v2-moe.md) — local Ollama serving `nomic-embed-text:v2-moe`
- [`vector-search-via-convex-vectorindex`](./vector-search-via-convex-vectorindex.md) — Convex `vectorIndex` is the vector store

## Process

- [`session-start-book-root-only`](./session-start-book-root-only.md) — read `~/tc/book/` root files only at session start; subfolders are other projects
- [`ledger-resume-protocol`](./ledger-resume-protocol.md) — `ledger.jsonl` + `resume` trigger word
