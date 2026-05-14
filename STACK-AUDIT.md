# STACK-AUDIT

Inventory of what's present in the byerag code repo at any given snapshot. Updated after each meaningful checkpoint. Reads the actual filesystem, not the spec — the truth lives in code, this file mirrors it.

## At baseline scaffold

Inherited from the substrate reference, stripped of domain-specific apps + tools + skills + tables.

### Present

- Monorepo skeleton: `apps/admin/`, `apps/user/`, `apps/backend/`, `packages/cli/`, `packages/react/`, `packages/q/`.
- Convex schema (`apps/backend/convex/schema.ts`): the tables enumerated in `SCHEMAS.md`, with `chats.app: 'admin' | 'user'` discriminator and `docs` table including `vectorIndex` on `embedding`.
- Convex http routes (`apps/backend/convex/http.ts`): auth, CLI manifest / exec / device / stream, agent stream-event + complete, Anthropic-shaped proxy at `/api/anthropic/*`.
- Agent script blob (`apps/backend/convex/agentScript.ts`): embedded into sandbox at boot; `AGENT_SKILLS_BY_APP = {"admin":{},"user":{}}` baseline (empty until tools land).
- Sandbox client (`apps/backend/convex/sandboxClient.ts`): present as substrate's E2B-shaped interface (`createSandbox`, `connectSandbox`). Implementation swap to Docker+gVisor is P1 scope.
- Per-chat proxy bearer scheme (`apps/backend/convex/messages.ts:anthropicProxy`): present, hardcoded upstream is `api.anthropic.com`. Swap to `api.kimi.com` is P1 scope.
- CLI infra (`packages/cli/`): manifest builder, dispatch, http, validate, prompt blocks, define-provider — all generic engineering, no domain leak.
- React chat lib (`packages/react/`): hooks, components, registries — generic.
- Tool registry stubs (`apps/backend/convex/tools/generated/`): empty `REGISTRY`, `TOOL_CALLERS`, `ToolTypes`. Codegen regenerates from real tool definitions in P3.
- Tool framework code (`apps/backend/convex/tools/_app/`): dispatch, cache, rate limit, mention resolver, skill-blob renderer, auth, CLI device flow, stream — all kept.

### Absent (P1 / later scope)

- Docker + gVisor sandbox client implementation.
- Kimi upstream URL constant in the proxy.
- ClamAV scan service.
- Ollama compose service.
- Real `docs` tool provider (`apps/backend/convex/tools/docs/`).
- Upload endpoint that writes to `docs` table + `_storage`.
- Admin app + user app UI beyond scaffolded sign-in page.

### Operator-local (not in repo)

- `.env` with concrete values (`AUTH_GOOGLE_ID`, `AUTH_GOOGLE_SECRET`, `CONVEX_SELF_HOSTED_URL`, `CONVEX_SELF_HOSTED_ADMIN_KEY`, `JWT_PRIVATE_KEY`, `JWKS`, `KIMI_API_KEY`, `BACKUP_AGE_PUBKEY`).
- `apps/backend/secrets/postgres_password.txt` + `apps/backend/secrets/convex_instance_secret.txt` (compose secrets files).
- Convex compose stack running on the host.
- gVisor (`runsc`) installed and registered as a Docker runtime.
- Ollama daemon running with `nomic-embed-text-v2-moe` pulled.
- ClamAV daemon.
- Host nftables ruleset applied per `network-bridge-rules.md`.
- Cron entries for daily backup + monthly restore drill.

### Added in baseline + gap-fill pass

- `compose.yml` (full stack with healthchecks + bridges).
- `apps/backend/sandbox/Dockerfile` (sandbox image source).
- Schema additions: `docs.{extractedText, lang, version, supersedes, supersededBy, deletedAt}` + nullable `docs.storageId` + `docChunks` table.

### P2 — docs corpus shipped

- `apps/backend/convex/docs.ts` — V8 queries/mutations for `docs` lifecycle: `findBySha256`, `findByFilename`, `insertRow`, `insertQuarantined`, `getForExtract`, `setExtracted`, `getForClassify`, `setPolicy`, `upload` action, `listMine`, `listShared`.
- `apps/backend/convex/docsUpload.ts` (`'use node'`): sha256 dedup → filename-version-conflict check → ClamAV TCP scan (zINSTREAM) → insertRow on clean / insertQuarantined on hit.
- `apps/backend/convex/docsExtract.ts` (`'use node'`): per-mime text extractor via sandbox `commands.run` (pdftotext + tesseract OCR fallback, pandoc, raw utf-8) + lang heuristic (CJK + Vietnamese diacritic regex).
- `apps/backend/convex/docsPolicy.ts` (`'use node'`): Kimi `/v1/messages` policy classifier; setPolicy mutation patches doc + schedules embed on approved.
- `apps/backend/convex/settings.ts`: get / seedDefaults / set; corpus_policy + agent_auto_assign_enabled keys seeded.
- `clamd` TCP scan service running on `clamd:3310` per `compose.yml`.

### P3 — agent tools shipped

- `apps/backend/convex/tools/docs/_provider.ts` declares the `docs` provider.
- `apps/backend/convex/tools/docs/{list,read,grep,diff,similar,conflict}.ts` — six tools, each a `defineQuery` or `defineTool`. ACL pushed into queries (mine-scope rows filtered by `owner === caller.email`; shared open).
- `apps/backend/convex/tools/generated/registry.ts` — codegen output from `x-codegen`.
- `apps/user/server/index.ts`: `cliProviders: ['docs']` so the agent-launch action ships the `docs` wrapper binary to the sandbox.

### P4 — embed + similar + conflict shipped

- `apps/backend/convex/docsEmbed.ts` (`'use node'`): sliding-window chunker (1600 / 200, sentence-boundary lookback) + Ollama `nomic-embed-text-v2-moe` via OpenAI-compat `/v1/embeddings` with `search_document:` / `search_query:` prefixes; per-doc centroid stored as `docs.embedding`.
- `apps/backend/convex/docs.ts` extensions: `getForEmbed`, `persistChunks`, `getRowsSnippet`, `getForConflict`; setPolicy schedules embed on approved.
- `apps/backend/convex/tools/docs/similar.ts`: `defineTool` action — `ctx.vectorSearch('docs','by_embedding',{filter,vector})` per scope, top-K cosine, runQuery `getRowsSnippet` for filename + 160-char snippet enrichment.
- `apps/backend/convex/tools/docs/conflict.ts`: `defineTool` action — Kimi semantic conflict scan w/ excerpt grep-verification per `auto-resolve-via-shared-kb-on-conflict.md`; sort factual→gap→wording; report `droppedHallucinated` count for excerpts that failed substring check.

### P7 — verify + harden in progress

- `apps/backend/scripts/smoke-supportiveness.ts` — 7-scenario harness driving the agent over scripted corpus seeds; deterministic auto-judge (tool-trace + keyword match + citation regex).
- `apps/backend/test-fixtures/supportiveness-scenarios.json` — committed manifest of scenarios (corpus seeds + prompts + expected tools + keyword/citation gates).
- `apps/backend/scripts/pull-test-corpus.ts` — Kimi-knowledge probe per `test-corpus-source-and-kimi-knowledge-probe.md`; reads `probe-candidates.jsonl`, runs Kimi w/ no doc context + Ollama embedding cosine; rejects docs Kimi already knows, writes accepted docs to `test-fixtures/docs/real/` (gitignored) + appends to `test-fixtures/probe-log.jsonl`.
- `apps/backend/scripts/smoke-agent-docs.ts` — end-to-end agent-uses-tools smoke (seed conflicting PTO docs → chat → assert agent invokes docs tools).

### Operator-local fixtures (gitignored)

- `apps/backend/test-fixtures/probe-candidates.jsonl` — operator-supplied probe candidates.
- `apps/backend/test-fixtures/probe-log.jsonl` — appended per probe run.
- `apps/backend/test-fixtures/docs/real/` — accepted real corpus snippets.
- `apps/backend/test-fixtures/supportiveness-evidence/` — per-scenario JSON evidence captures.
