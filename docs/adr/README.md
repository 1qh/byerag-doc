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
- [`role-on-user-profile`](./role-on-user-profile.md) — role stored on `userProfiles.role`; bootstrap admin via env; no network/VPN gating

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
- [`admin-scan-override`](./admin-scan-override.md) — admin-only force-upload for ClamAV-rejected files, gated by typed `OVERRIDE` confirmation, audit `severity='high'`
- [`concurrency-and-active-context-token`](./concurrency-and-active-context-token.md) — single active tab per user gating
- [`timestamps-and-timezone`](./timestamps-and-timezone.md) — UTC epoch ms in DB, ISO-Z in ledger, local in UI
- [`owner-id-canonical-email-lowercase`](./owner-id-canonical-email-lowercase.md) — owner string == lowercase email
- [`ui-error-surfacing`](./ui-error-surfacing.md) — inline / toast / banner per error class
- [`long-running-tool-call-policy`](./long-running-tool-call-policy.md) — per-tool deadlines + streaming exec
- [`multilingual-corpus-handling`](./multilingual-corpus-handling.md) — single embedding model handles ~100 langs + `docs.lang` for display
- [`schema-spec-fields-canonical`](./schema-spec-fields-canonical.md) — identity / time / enum / index naming conventions

## Assessment tests

- [`assessment-test-overview`](./assessment-test-overview.md) — top-level invariants + sibling-ADR index
- [`topic-clustering-plan-b`](./topic-clustering-plan-b.md) — flat agent-clustered topic list; no hierarchy
- [`question-generation-pipeline`](./question-generation-pipeline.md) — async gen on approved doc; 10 per doc; Vietnamese; conflict scan
- [`question-review-queue`](./question-review-queue.md) — admin actions; bulk-approve; conflict + cap-swap pairs
- [`question-pool-soft-cap-50`](./question-pool-soft-cap-50.md) — 1-in-1-out at cap; admin stretch allowed
- [`assessment-test-attempts`](./assessment-test-attempts.md) — 5 Qs / 100% pass / open-book / shuffle / discard-on-close
- [`assessment-assignments`](./assessment-assignments.md) — admin to role=user / real-time / persistent badge / un-assign nuke
- [`re-arm-on-substantive-corpus-update`](./re-arm-on-substantive-corpus-update.md) — admin substantive flag invalidates assigned-passes
- [`chat-agent-training-tools`](./chat-agent-training-tools.md) — read-only own-data CLI surface

## Dashboard

- [`dashboard-admin-landing`](./dashboard-admin-landing.md) — `/admin/dashboard` as default landing route
- [`dashboard-top-strip`](./dashboard-top-strip.md) — three live tiles: active/total users, cost cycle, docs in corpus
- [`dashboard-cost-cycle`](./dashboard-cost-cycle.md) — 5th-to-5th cycle, monthly chart, per-(user, model) pivot
- [`dashboard-gradebook`](./dashboard-gradebook.md) — per-(user, topic) matrix with glyphs + drill-ins
- [`costrecords-table`](./costrecords-table.md) — per-(owner, model, dayKey) aggregation backing cost analytics
- [`departments`](./departments.md) — HR/Sales/IT tag on role=user; dashboard-filter affordance only

## Agent auto-assign

- [`agent-auto-assign-cron`](./agent-auto-assign-cron.md) — daily cron + manual enable flag; signal (i) only; no rate limit; aggregate audit per run

## Production deployment

- [`prod-deployment-pattern-dokploy`](./prod-deployment-pattern-dokploy.md) — Dokploy on operator VM; git-push auto-deploy; read-only API access from agent; concrete creds in agent memory

## Test corpus

- [`test-corpus-source-and-kimi-knowledge-probe`](./test-corpus-source-and-kimi-knowledge-probe.md) — post-cutoff real docs + Kimi-knowledge probe + fabricated edge-case fixtures

## Process

- [`session-start-book-root-only`](./session-start-book-root-only.md) — read `~/tc/book/` root files only at session start; subfolders are other projects
- [`ledger-resume-protocol`](./ledger-resume-protocol.md) — `ledger.jsonl` + `resume` trigger word
