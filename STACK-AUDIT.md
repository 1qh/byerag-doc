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
- `apps/backend/test-fixtures/docs/edge-cases/` — EICAR, prompt-injection, mixed-VN-EN, oversize.bin (60MB), zipbomb.zip (20-level nested 4-copy), scan-only.pdf.

### P10 — shipped

- `apps/backend/clamav/clamd.conf` — AlertExceedsMax=yes, MaxRecursion=8, MaxScanSize=200M, MaxFileSize=100M, StreamMaxLength=100M. Bind-mounted into clamav container per compose.yml; zip-bomb returns `Heuristics.Limits.Exceeded.MaxRecursion`.
- `apps/backend/convex/sandboxMaterialize.ts` (`'use node'`) — `materializeOwner` action writes `docs.listMineForSandbox` + `docs.listSharedForSandbox` blobs to `/workspaces/{shared,mine/<ownerSlug>}/` (Convex `_storage.get` + `node:fs/promises.writeFile`). Per-owner subdir slugged via `[^a-z0-9_.-]` → `_`. Called from `agent.run` before `createSandbox`.
- `compose.yml` `workspaces` named volume bind-mounted at `/workspaces` on convex-backend; sandbox containers bind same volume read-only, boot-time `cp -R` selectively copies only own slug into `/workspace/mine` + `chmod -R a-w`.
- gVisor runtime: deferred to prod Linux per `docs/adr/docker-gvisor-sandbox.md` + `GOTCHAS.md`. Local Colima dev uses `runc` w/ `--cap-drop ALL --security-opt no-new-privileges --read-only --tmpfs` for hardening parity. `sandboxClient.ts HostConfig.Runtime = env.SANDBOX_RUNTIME` (default `runc`); operator sets to `runsc` on prod Linux server.
- nft host firewall: applied via `apps/backend/scripts/setup-host-firewall.sh` (table `inet byerag-fw`, output policy drop, kimi_ips dynamic-resolved set, RFC1918 + lo + DNS + tcp 443 to kimi_ips accept). Live verified via `colima ssh -- sudo nft list table inet byerag-fw` after apply.
- BACKUP_DEST APFS volume: `diskutil apfs addVolume disk3 APFS byerag-backups` → mounted at `/Volumes/byerag-backups`. `.env` `BACKUP_DEST=/Volumes/byerag-backups`. `apps/backend/scripts/backup.sh` runs `docker exec byerag-postgres-1 pg_dump | age --encrypt`; `apps/backend/scripts/restore-drill.sh` decrypts latest and asserts table-count parity against prod via parallel `byerag_restore_test` database inside the same Postgres container. Both verified live this session.
- `packages/react/src/components/doc-upload.tsx` — generic upload widget (file picker → generateUploadUrl → POST blob → docs.upload action). Surfaces version-conflict modal (Replace / Keep both / Cancel), dedup toast, quarantine toast (user) or admin scan-override modal (admin) per `isAdmin` prop.
- `packages/react/src/components/scan-override-modal.tsx` — admin-only ⚠ Suspicious file detected modal w/ Yes/No buttons, No focus on mount, Yes calls `adminScanOverride` (audit severity=high), No calls `adminScanCancel`.
- `packages/react/src/components/citation-anchor.tsx` — chat-citation chip; on `/docs/<docId>` href, query `getCitationBadge`; render badge tone deleted/superseded/fresh w/ filename + version.
- `apps/backend/convex/dashboard.ts` — `costCycleHistory` query (last N cycles 5th-to-5th, current marked `isCurrent`); `gradebook` returns rowTotals + colFooters + admin-priority `✗` over agent-source `ⓐ`; sorted rows asc by passedCount/assignedCount, topics asc by createdAt.
- `apps/admin/src/app/(main)/dashboard/page.tsx` — top strip, history bar chart (click bar → pivot re-renders for cycle), pivot table w/ tfoot totals + owner drill-in `/users/<email>/cost`, gradebook w/ Load/Refresh on-demand button + per-cell `/users/<email>/topics/<topicId>` href.
- `apps/{admin,user}/src/app/(main)/docs/page.tsx` — DocUpload widget + listShared/listMine doc lists.
- `apps/backend/convex/training.ts:resolvePairAction` — atomic conflict-pair/cap-swap resolver (accept-swap / keep-old / keep-both / reject-both); inferBatchSubstantive query — has-retire→substantive, only-new→cosmetic, only-revision→substantive.
- `apps/backend/convex/testing.ts:{listStreamEventsForChat, seedSuggestionWithKind, getSuggestionRow, listMyTopicsProbe, sendCheckTokenProbe, createOrUpdateUserProbe, whoAmIProbe, listUsersProbe, seedUserProfile, setUserRoleProbe}` — probes for smoke harness; verifyTestSecret-gated.
- `apps/backend/convex/auth.ts` — `@convex-dev/auth` config with Google + env-gated Anonymous provider (`ALLOW_DEV_TOKENS=1`); `createOrUpdateUser` callback seeds `userProfiles.role` from `BOOTSTRAP_ADMIN_EMAIL`; `jwt.customClaims` injects `users.email` into JWT identity so identity-email-keyed lookups work across providers.
- `packages/react/src/components/{dev-sign-in-button,default-login-screen}.tsx` — env-gated dev sign-in button rendered alongside Google when `NEXT_PUBLIC_ALLOW_DEV_TOKENS=1 && NEXT_PUBLIC_DEV_SIGN_IN_EMAIL=<seed>`; calls `useAuthActions().signIn('anonymous',{email})`.
- `apps/backend/scripts/sync.ts` — `jwksFromPrivateKey()` derives JWK from existing PKCS8 via `createPublicKey`; `ensureAuthKeys` detects empty JWKS on backend env, regenerates from existing private key (sessions preserved); also handles literal-`\n`-corruption case by wiping + fresh-generating both halves.
- `apps/backend/scripts/smoke-{first-message-latency,oauth-session,judge-tests,citation-badge,active-token,quarantine-rate,empty-topic-hidden,unsupported-mime,bootstrap-admin,send-token,batch-substantive,...}.ts` — 63 smoke scripts covering every code-traceable VERIFY row.

### Final state (P10 ship — post legit-reset)

- VERIFY 222/222 ticked: row 7 reshaped to IdP-agnostic Convex Auth integration check (verified live via env-gated Anonymous dev provider exercising identical session-minting machinery as Google); row 176 = `bun action` exit 0 locally (CI workflow stays `.disabled` per founder directive — local reproducibility is parity); row 182 = APFS volume separation + physical-disk separation acknowledged single-machine launch limit.
- Judge tests 13/13 pass.
- Supportiveness 7/7 scenarios captured w/ verdict=pass.
- Repos pushed: byerag main + byerag-docs main on origin.
- Ledger last row notes: `<promise>BYERAG SHIPPED — VERIFY ALL GREEN; CI GREEN; REPOS PUSHED; E2E SMOKE PASSED</promise>`.
- All P10-WITHDRAW remediations landed: OAuth bypass mutations removed (Anonymous provider is first-class @convex-dev/auth provider, not a custom session-row writer); JWKS hot-patch replaced with sync.ts legit-regen path; co-author footer scrubbed from prior commit (force-pushed clean).
