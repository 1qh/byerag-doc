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
- [`clamav-via-convex-action`](./clamav-stateless-scan-service.md) — Convex action talks to clamd directly; no wrapper service
- [`ollama-nomic-embed-v2-moe`](./ollama-nomic-embed-v2-moe.md) — local Ollama serving `nomic-embed-text:v2-moe`
- [`vector-search-via-convex-vectorindex`](./vector-search-via-convex-vectorindex.md) — Convex `vectorIndex` is the vector store

## Operations

- [`sandbox-image-and-cli-delivery`](./sandbox-image-and-cli-delivery.md) — sandbox image content + CLI injected at agent-run
- [`text-extraction-by-mime`](./text-extraction-by-mime.md) — per-mime extractor table + OCR fallback
- [`embedding-chunking-strategy`](./embedding-chunking-strategy.md) — docChunks table + sliding window + per-doc centroid
- [`kimi-cost-rates-and-reservation`](./kimi-cost-rates-and-reservation.md) — rates table + reservation math + defaults
- [`system-prompts`](./system-prompts.md) — per-app prompt shape + substitution rules
- [`ci-and-pre-commit-gates`](./ci-and-pre-commit-gates.md) — lefthook + universal CI + drift/leak checks
- [`backups-pg-dump-and-restore-drill`](./backups-pg-dump-and-restore-drill.md) — daily age-encrypted pg_dump + monthly restore drill
- [`local-dev-loop`](./local-dev-loop.md) — `bun dev`, port map, hot reload
- [`network-bridge-rules`](./network-bridge-rules.md) — sandbox-egress bridge + iptables + DNS isolation
- [`audit-retention-and-purge-cron`](./audit-retention-and-purge-cron.md) — 90d retention + nightly purge

## Data model details

- [`doc-versioning-and-deletion-cascade`](./doc-versioning-and-deletion-cascade.md) — version chain + soft-delete + scheduled hard-purge
- [`upload-dedup-and-version-prompt`](./upload-dedup-and-version-prompt.md) — duplicate content auto-reject + version-conflict modal (Replace / Keep both / Cancel)
- [`policy-relevance-classifier`](./policy-relevance-classifier.md) — LLM classifier on every upload; admin-tunable policy text; quarantine queue for review
- [`concurrency-and-active-context-token`](./concurrency-and-active-context-token.md) — single active tab per user gating
- [`timestamps-and-timezone`](./timestamps-and-timezone.md) — UTC epoch ms in DB, ISO-Z in ledger, local in UI
- [`owner-id-canonical-email-lowercase`](./owner-id-canonical-email-lowercase.md) — owner string == lowercase email
- [`ui-error-surfacing`](./ui-error-surfacing.md) — inline / toast / banner per error class
- [`long-running-tool-call-policy`](./long-running-tool-call-policy.md) — per-tool deadlines + streaming exec
- [`multilingual-corpus-handling`](./multilingual-corpus-handling.md) — single embedding model handles ~100 langs + `docs.lang` for display
- [`schema-spec-fields-canonical`](./schema-spec-fields-canonical.md) — identity / time / enum / index naming conventions

## Process

- [`session-start-book-root-only`](./session-start-book-root-only.md) — read `~/tc/book/` root files only at session start; subfolders are other projects
- [`ledger-resume-protocol`](./ledger-resume-protocol.md) — `ledger.jsonl` + `resume` trigger word
