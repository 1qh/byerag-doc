# GOTCHAS

Evolving per-topic gotcha placeholder. Each section accrues lessons as the build hits friction. Per `book/PHILOSOPHY.md` "Capture gotchas at every milestone".

When a new gotcha lands: append one paragraph under the most relevant section (what surprised, why it surprised, what to do next time). No append-only "recent lessons" bucket — merge into topic.

## Convex self-host

- **Postgres 18+ breaks default volume mount path** — PGDATA moved to `/var/lib/postgresql/<MAJOR>/docker`. Stick to `postgres:17-alpine`. Surfaced 2026-05-14 P0 boot.
- **POSTGRES_URL must NOT include `/dbname`** — Convex backend errors `cluster url already contains db name`. Use `postgres://user:pass@host:port`; Convex derives db name from `INSTANCE_NAME` (dashes→underscores). Surfaced 2026-05-14 P0 boot.
- **`docker compose restart` ignores `.env` changes** — env substitution happens at container create time. Use `docker compose up -d --force-recreate <service>`. Surfaced 2026-05-14 P0 boot.
- **macOS `localhost` resolves `::1` first; Convex binds `127.0.0.1:3210` (v4)** — Node 20 / browser fetch to `http://localhost:3210` hits `::1`, ECONNREFUSED. All `.env` URLs (`NEXT_PUBLIC_CONVEX_URL`, `CONVEX_SELF_HOSTED_URL`, `CONVEX_SITE_URL`) must be literal `127.0.0.1`. SSH port-forwards forward v4 only — same trap on remote dev machines. Surfaced 2026-05-16.
- **SSH dev forwards need 4 ports** — `3001` admin, `3003` user, `3210` Convex API, `3211` Convex site. Missing 3210/3211 = browser WebSocket sync fails silently (login screen sits "Loading…"). Surfaced 2026-05-16.
- **OCC hot-row contention on `chatRuntime` + `rateLimits` + `costRecords`** — per-stream-event `incrementEventCount` patch on the chat's single chatRuntime row + per-event `checkRateLimit` mutations against the same owner row produced "Documents read/written too many times" mid-stream, surfacing as client toast "an unexpected error occurred". Fix triad in `apps/backend/convex/messages.ts`: (1) drop per-event eventCount patch — use monotonic stream `seq` against `STREAM_EVENT_HARD_CAP` instead; (2) remove the two per-event stream rate-limit checks; (3) wrap `checkRateLimit` in `safeRateLimit` that catches OCC errors and returns `true` (fail-open). Hard-cap stays exact via seq; rate-limit fail-open is acceptable for stream firehose. Surfaced 2026-05-16.
- **Full-table scan inside ratelimit-like hot path times out at 1s** — `docs.countRecentQuarantines` scanned all `docs` rows. Switch to indexed `by_sha256_scope_owner` `.take(50)` then filter in JS. Surfaced 2026-05-16.
- **Self-host deploy is environment-fragile; the 4s analyze isolate timeout is NOT configurable** — `convex --help` exposes no flag. `start_push` fails `400 InvalidModules: Function execution timed out (maximum duration: 4s)` whenever the V8 module-analyze loses CPU to the cold node-executor bundle (28–62s) of the many `'use node'` modules. Reliable ONLY when (a) node bundle is warm and (b) the box has CPU headroom. Root cause this session was host saturation: Colima had **4 vCPU of a 10-core host**, starved by co-tenant containers (`map-kafka` ~1.5 core/7.4 GB, `k3d-truecare` ~1 core/4 GB) + host procs (a 365% Java proc, 24 GB FSEvents) → host load 80, 0% idle. Deploy went green in 40s the instant the box was freed. Surfaced 2026-05-17.
- **NEVER cold-restart convex-backend or `docker compose down -v` casually** — that cold-wipes the warm node-executor bundle (forcing the 60s cold path), wipes all deployed functions (even `auth:signIn` → nobody can sign in), and clears `costRecords`/auth tables (→ stale browser sessions, below). The half-measure `docker compose restart convex-backend` is the WRONG recovery; it caused the deploy outage this session.
- **The backend had no `dev` script → no warm convex watcher ever ran** — `apps/backend/package.json` was missing `dev`, so `turbo dev` only started Next; every push was a fresh cold `convex dev --once`. Fix: `apps/backend` `dev` = `bun run build-agent && bun run codegen && bunx convex dev` (persistent watch — one analyze, stays warm). Canonical loop per `local-dev-loop.md`; `--once` cold-starts every invocation.
- **Do NOT loop-spam `convex dev --once`; do NOT deploy concurrently with app use** — repeated/parallel pushes saturate the single isolate pool and starve runtime UDFs: `auth:signIn` → `auth.js:store` hit the 1s UDF budget and sign-in broke. Deploy = single, sequential, box otherwise quiet, no users active.
- **`.env` JWT-regen corruption** — `sync.ts` JWT regen can write a multi-line PEM unquoted/duplicated (orphan `MIIE…` body block with no `KEY="-----BEGIN…` header), making `.env` unparseable → `docker compose` env error + convex push fails. Fix: clear `JWT_PRIVATE_KEY=` and `JWKS=` lines, re-run `bun sync` (regenerates a matched pair). Surfaced 2026-05-17.
- **Stale browser session after any wipe is THE cascade root cause** — DB wipe destroys `users`/auth tables + JWT regen invalidates the cookie. Every identity-gated query returns `null`/`[]` → docs won't preview, `/training` empty, "Start does nothing", sign-in timeouts — all the SAME bug. Fix (founder action): sign out + **clear site data for the origin** (orphaned token) + sign in fresh. Agent-drivable path: env-gated dev sign-in (below).
- **Convex hard-caps a single document value at 1 MiB — extracted text overflows it** — `docs.setExtracted` patching a 3.25 MiB `extractedText` inline threw `Value is too large (3.25 MiB > maximum size 1 MiB)`, the `docsExtract` action died, and the doc stuck `policyStatus='pending'` forever (no classify/embed/questions; agent can't read it) — every sufficiently large doc silently failed ingestion. Canonical fix: extraction writes the FULL text to a `_storage` text blob (`docs.extractedTextStorageId`); `docs.extractedText` keeps only a bounded prefix (`EXTRACT_INLINE_MAX_CHARS`) covering all prefix-only readers (classify ≤4K, gen ≤12K, conflict ≤50K, snippets). Chunking/embedding reads the blob so vector/`docs similar` covers the whole doc. Follow-up (tracked): migrate `docs read`/`grep`/`diff` (`defineQuery`, can't `ctx.storage.get` bytes) to blob-reading actions; until then they operate on the inline prefix (no hard error; semantic search unaffected). Per `text-extraction-by-mime.md` + `SCHEMAS.md`. Surfaced 2026-05-17.

## Ollama embedding

- **Native Mac Ollama, NOT dockerized** — `ollama serve` runs on host; Convex container reaches via `host.docker.internal:11434`. Containerized Ollama wastes RAM + duplicates model file. Surfaced 2026-05-14 P0 boot.
- **Model slug `nomic-embed-text-v2-moe`** (dashes, single segment). NOT colon-tag form `nomic-embed-text:v2-moe` — that's wrong; non-existent tag of a different base model. Library URL is authoritative. Surfaced 2026-05-14 P0 boot when pull returned `Error: pull model manifest: file does not exist`.
- **Use OpenAI-compat `/v1/embeddings`** endpoint, NOT native `/api/embed` — portable across providers. Same model name, OpenAI-shape request/response.

## ClamAV scan

- **`clamav/clamav:latest` is amd64-only** — no arm64 manifest. Use `clamav/clamav-debian:latest` (multi-arch) on Apple Silicon. Surfaced 2026-05-14 P0 boot.

## Docker + gVisor

- **`node:20-slim` preinstalls a `node` user at UID 1000** — `useradd -u 1000 agent` fails with `useradd: UID 1000 is not unique`. Drop the existing user first: `RUN userdel -r node 2>/dev/null || true` before `useradd -m -u 1000 -s /bin/bash agent`. Surfaced 2026-05-14 P0 sandbox image build.
- **Local dev uses plain `runc`, not `runsc`** — gVisor deferred to prod Linux per `docker-gvisor-sandbox.md`. Colima on Mac doesn't ship `runsc`; trying `--runtime=runsc` locally fails. Local hardening relies on `--cap-drop ALL --security-opt no-new-privileges` instead.
- **Sandbox runs as `User: agent` (uid 1000) — cannot write to `/usr/local/bin`** — any CLI wrapper / symlink the agent-run action places must land under a path the `agent` user owns. `/home/agent/.bun/bin/` is part of the container's default `PATH` (`/home/agent/.bun/bin:/usr/local/bin:/usr/bin:/bin`), is `agent`-owned, and is the canonical home for provider wrappers (`docs`, `training`, …) per `sandbox-image-and-cli-delivery.md`.
- **`SANDBOX_PATH` env-override does not always reach the Claude SDK Bash subshell** — the Claude Agent SDK spawns child Bash processes whose `PATH` is inherited from the container default, not always from the parent agent process's env-set `PATH`. The defense: place CLI wrappers at a path already on the container default `PATH` (`/home/agent/.bun/bin/`); don't rely on the env override propagating.
- **POSIX wrapper, not symlink, for CLI provider binaries** — a symlink at `/home/agent/.bun/bin/<provider>` pointing to `/home/agent/cli.mjs` lets the next `printf … > path` overwrite cli.mjs through the symlink. Use a 45-byte wrapper script (`#!/bin/sh\nexec node /home/agent/cli.mjs "$@"`) written via `printf` after `rm -f` of any prior file; chmod +x in the same `&&` chain.
- **App `cliProviders: []` ships zero wrappers to sandbox PATH** — `sandboxLaunch.prepareSandboxLayout` writes wrapper scripts to `/home/agent/.bun/bin/<provider>` ONLY for providers listed in the app's `AppConfig.cliProviders`. Admin app was created with `cliProviders: []`; agent's first turn said "The `docs` CLI wasn't available on PATH" because nothing was written. Admin needs `['docs', 'training']`; user needs at least `['docs']`. Surfaced 2026-05-16. **Defense**: any new app must explicitly enumerate every provider its system prompt invokes.
- **Docker exec `sh -lc` is a login shell — strips the agent process's PATH** — `sh -l` re-sources `/etc/profile` which sets `PATH=/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games`, overriding the `PATH` env passed via Docker exec's `Env` field. The agent process loses `/home/agent/.bun/bin` from PATH; the Claude SDK's Bash subshells then can't find the provider wrappers. Canonical: `sh -c` (non-login) for exec invocations — preserves the Env-set PATH end-to-end. Captured in `sandboxClient.ts:execInside`.
- **`byerag-sandbox:latest` is built separately, NOT in `compose.yml`** — `docker system prune` (or any image prune) deletes it; extraction (`docsExtract`) + chat agent then fail silently with `docker POST /containers/create 404: No such image`. Symptom cascade: `_scheduled_functions` shows `docsExtract.extract` all `failed` → docs stuck `policyStatus='pending'` → no policy classify → no question generation → `/test-questions` empty. Fix: `docker build -t byerag-sandbox:latest apps/backend/sandbox/` (drop nft firewall first — see below — then reapply). Defense: bring-up routine must rebuild this image after any prune/`down -v`. Surfaced 2026-05-17.
- **Host nft firewall (Kimi-only egress) blocks `ghcr.io`/`docker.io` image pulls + npm** — `docker compose up`/`docker build` time out on image resolve, and the Convex node-executor can't fetch deps. Documented provisioning pattern: `colima ssh -- sudo nft delete table inet byerag-fw` → pull/build → reapply the `byerag-fw` table (multi-line `nft -f -` heredoc; the compact one-liner form errors on `}`). Surfaced repeatedly 2026-05-16/17.

## Kimi proxy

- **`stop_reason="tool_use"` after the last tool call leaves the result empty** — Kimi sometimes ends an agent turn with a tool_use block and never emits the follow-up text turn that would synthesize the answer; the SDK records `result.result = ""` and the chat shows no answer body. Mitigations baked into `system-prompts.md`: explicit "final-answer protocol" mandating a plain-text response after the tool chain; smoke harness asserts non-empty assistant text + keyword + citation match per scenario.
- **Stream-event firehose exceeds default rate-limit caps** — Kimi emits per-token deltas during streaming; an agent that composes 4+ tool calls in one turn produces 1K–8K stream events. Default per-chat cap (300/min) and per-owner cap (900/min) starved the chat of events and corrupted `complete`-time message reconstruction. Canonical caps for `/api/stream/event`: per-chat 8_000, per-owner 20_000, refilled across a 60s window. Captured in `SECURITY.md` rate-caps table.
- **Kimi 429 → eternal spinner, no answer** — when Kimi hard-throttles, the SDK retries with escalating backoff and on exhaustion the agent script threw. The catch block POSTed only an `error` event and NOT `/api/stream/complete`, so `chats.streaming` stayed `true` forever — infinite "Thinking…" spinner with no terminal message. Liveness can't rescue it: retry-notice events keep arriving so the 90s zero-events check never trips. Canonical fix: the `run.ts` catch path posts the redacted error event AND THEN `/api/stream/complete` (same termination as success). Proxy also logs `proxy.upstream.429` distinctly (was opaque `non-sse-error`). Per `streamevents-reactive-pipeline.md`. Surfaced 2026-05-17 (chat owned by `lqhxyz@gmail.com`, observed via `/tmp` convex log refund/retry churn).

## Ollama embedding

(none yet)

## ClamAV scan

(none yet)

## Agent SDK

- **`@anthropic-ai/claude-agent-sdk` 0.3.x dropped `unstable_v2_createSession` / `unstable_v2_resumeSession`** — `sandbox/run.ts` imports those names; bun throws `SyntaxError: Export named 'unstable_v2_createSession' not found` on agent boot → pgid file never written → orchestrator surfaces `agent failed to start (pgid not written)` after 30s. Sandbox is auto-killed so the trace is invisible. Defense: `apps/backend/sandbox/package.json` pins `@anthropic-ai/claude-agent-sdk` to `0.2.141` (last `unstable_v2_*` carrier). Refactor `run.ts` to the new `query()` API before un-pinning; SDK 0.3+ surface is `query`, `forkSession`, `resumeSession` (top-level, not `unstable_v2_*`), `listSessions`, etc. Bun's import-failure path = silent in agent.log because the orchestrator's outer `nohup … >/dev/null 2>&1 &` clobbers the inner `>$logFile` redirect — when diagnosing, reproduce with `docker run --rm --entrypoint sh byerag-sandbox:latest -c 'setsid bun run /home/agent/run.ts 2>&1'` + minimal env. Surfaced 2026-05-16 after fresh `docker compose down -v` + image rebuild pulled SDK 0.3.143.

## pm4ai + lintmax

- **`bun run fix` auto-fix can strip `await` from chained Convex `.first()` calls** when the value variable looks unused at the syntactic level (linter's "remove redundant await on non-Promise" pass triggers on the misclassification). The Convex query chain returns a thenable; dropping `await` leaves the variable as `{kind:'first', ...}` query token, and downstream `.patch(existing._id, ...)` blows up with `Must provide arg 1 'id' to 'patch'`. Defense: keep the variable in a clear `await x.first()` shape; add a use of the variable in the same expression (e.g. `if (existing?._id)`) when the linter trips again. Captured 2026-05-14 from turn-budget smoke regression in `testing.ensureChatRuntime`. **Permanent defense**: prefer `.collect()[0]` over `.first()` when the result feeds a subsequent `.patch()` — the auto-fix consistently strips `await` from `.first()` even with typed annotation, but never strips it from `.collect()`. Concretely: `const rows = await ctx.db.query(...).withIndex(...).collect(); const existing = rows[0]`.

## Next.js 16 + Turbopack

- **`app.config.ts` with JSX fails build** — Next 16 + Turbopack will not parse JSX in `.ts`. Any app-config file that returns `sidebarSlotAboveHistory: <Component/>` must be `.tsx`. Surfaced 2026-05-16 when adding `AdminSidebarNav` to admin app.
- **`PaneProvider` must wrap children inside `DefaultProviders`** — `packages/react/src/components/message-text-part.tsx` calls `usePane()`; without provider it throws "usePane must be used inside <PaneProvider>" the moment a chat renders a message. Canonical wrap order in `packages/react/src/next/default-providers.tsx`: `TooltipProvider > PaneProvider > VerbosityProvider`. Surfaced 2026-05-16.
- **Route group `(main)` shared layout includes chat shell** — pages that should stand alone (dashboard, /test-questions, /docs viewer) must live OUTSIDE `(main)` and ship their own `layout.tsx` (Auth gate + `AdminSidebarNav` aside, no chat shell). Surfaced 2026-05-16. **User app had this same defect** — every user route was under the chat shell; non-tech employees saw a chat composer under `/training` `/docs`. Both apps need the standalone-group pattern; the user app is the MORE non-tech audience and was missed first.
- **Doc preview blank — CSP blocked the cross-origin Convex blob** — `DocViewer` rendered extracted text as a `<pre>` (raw markdown source, broken for PDF/docx). Switched to mime-routing (pdf→`<iframe>`, image→`<img>`, markdown→`MessageResponse`, office→extracted-text+"formatting not preserved"+download, code/text→`<pre>`) and added a blob `url` to `docs.read`. But the embed stayed blank on BOTH apps: the shared CSP in `packages/react/src/next/proxy.ts` had `default-src 'self'` and NO `frame-src`/`object-src`, so the Convex storage origin (`NEXT_PUBLIC_CONVEX_URL`, e.g. `http://127.0.0.1:3210`) was blocked — console: *"Loading plugin data … violates Content Security Policy directive default-src 'self' … blocked"*. Fix: add `frame-src`/`object-src` and extend `img-src` with `${convexOrigin} https://*.convex.cloud https://*.convex.site`. Also: `<object>` collapses to 0px under flex-only sizing and headless Chromium can't render the PDF plugin — use `<iframe>` with an explicit height (`h-[80vh]`), verify in a real browser screenshot not the a11y tree. Surfaced 2026-05-17.
- **shadcn here is base-ui, NOT Radix** — components use the `render` prop, not `asChild`. `<Button asChild>` leaks `aschild` to the DOM (React unknown-prop error) and does not compose. And base-ui Button `render`'d as a non-`<button>` warns (`nativeButton`). For a link styled as a button use `<Link className={cn(buttonVariants({variant}))}>`, not `<Button render={<Link/>}>`. `DropdownMenuTrigger`/`AlertDialog…` etc. all use `render={<X/>}`. Surfaced 2026-05-17.
- **`react-pdf` ships its own `pdfjs-dist`; never pin `pdfjs-dist` separately** — adding `pdfjs-dist` as an explicit dep in `packages/react/package.json` causes the worker version to diverge from `react-pdf`'s bundled main thread (`API version 5.4.296 does not match Worker version 6.0.227`). Canonical: depend only on `react-pdf`; resolve worker via `new URL('pdfjs-dist/build/pdf.worker.min.mjs', import.meta.url)` so it loads the same transitive copy. Wrap the consumer in `dynamic(() => import('./pdf-preview'), { ssr: false })` — pdf.js touches `DOMMatrix` at module top and crashes Next 16 SSR otherwise. Iframe-based fallback for plain blob URLs works in headed Chromium but not headless; for in-app preview canvas-rendered via `<Document>`/`<Page>` is the legitimate path.
- **`_clientMiddlewareManifest.js` console warning: "MIME type ('application/json') is not executable, strict MIME type checking is enabled"** — Next 16 emits the file with `.js` extension but `Content-Type: application/json` and ships `X-Content-Type-Options: nosniff` from its built-in static handler (NOT from our `proxy.ts` — its matcher excludes `_next/static`). With `proxy.ts` present (load-bearing for CSP), Next emits the manifest in `_document` as `<script src=…>`; browser refuses to execute → console warning. No functional impact: the manifest is a route-matcher data structure, not runnable JS; CSP still applies via `proxy.ts` server-side. Cannot be suppressed without removing `proxy.ts` (which would drop the CSP). Upstream Next 16 quirk; live with the dev-only warning until upstream serves the manifest with `Content-Type: text/javascript`.
- **Turbopack panic "Next.js package not found" after a mid-session `bun i`** — Next 16 dev server holds module-graph references to the pre-install `next` package; if `bun i` runs underneath (e.g. workspace dep churn for `react-pdf` dedup), every subsequent route compiles to a 500 with body `{"err":{"message":"Panic in async function"}}`. Stack from `/tmp/next-panic-*.log`: `subscribe_issues_and_diags_operation → pick_endpoint → entrypoints_without_collectibles_operation → AppProject::routes_with_filter → directory_tree_to_entrypoints_internal → directory_tree_to_loader_tree → Next.js package not found`. The browser sees a `__NEXT_DATA__` payload with `pageProps.statusCode:500` — easy to misread as a page-code bug. Symptom this session: admin `/dashboard` + `/policy` 500 while other admin routes also failed (only `/docs` rendered, from an earlier compiled chunk). Canonical fix: kill the `next dev` wrappers (both apps) + `rm -rf apps/<app>/.next apps/<app>/.turbo` + restart via `bun --filter=<app> dev`. After restart all routes return 200. The cache nuke alone is not enough — the running process must restart.
- **`NEXT_PUBLIC_*` only re-reads on Next dev restart** — apps run `bun --env-file=../backend/.env next dev`; env loaded at process start. After adding e.g. `NEXT_PUBLIC_ALLOW_DEV_TOKENS=1`, must restart the Next servers. The env-gated dev sign-in (Anonymous provider) needs BOTH `ALLOW_DEV_TOKENS=1` (server, auth.ts provider list) AND `NEXT_PUBLIC_ALLOW_DEV_TOKENS=1` (client, renders the dev sign-in). It is the spec-sanctioned no-Google session path (and the only way to drive an authed session via Playwright). Surfaced 2026-05-17.

## Convex schema migrations

(none yet)

## CLI builder defaults

- **`arg.*` with `default:` auto-marks the arg optional + auto-substitutes the default at handler entry.** Spec stores `default` alongside `optional`; if `optional` is omitted, `optional := default !== undefined`. `buildFullArgs` wraps the validator with `v.optional` when optional, so the Convex action accepts `undefined`. `applyDefaults` runs inside `defineTool`/`defineQuery`/`defineMutation` before the user handler — so handler sees the value as `Infer<V>`, never `| undefined`, and call sites do NOT write `args.x ?? <default>`. `validateArgs` (CLI-side argv parsing) substitutes the default for empty input too. Canonical call shape: `arg.number({ default: 50, description: '…' })` — no `optional: true`, no `??` in handler.

## Next.js route group bouncing

- **`default-main-layout` must not redirect `/chat/:id` based on sidebar-list membership** — `useChatList` is paginated and lags fresh chats by a render or two; gating routing on `chats.some(c => c._id === activeChatId)` redirects every brand-new chat to `/` until the list catches up. Canonical guard: `useQuery(api.chats.status, {chatId})` and redirect only when `chatStatus !== undefined && chatStatus.title === ''` (server-authoritative not-found). This unblocks "Ask about this" + any hard-nav landing on `/chat/<id>`; pair with `router.push` (the `globalThis.location.assign` workaround is unnecessary once the guard is correct).

## Vector search filter pushdown

- **Matryoshka shorter-dim queries against a fixed-dim index**: Convex `vectorIndex` declares `dimensions: 768` and accepts only that exact length. Matryoshka prefix queries (256 / 512) are realized by truncating the query vector to the first N dims and zero-padding the remainder back to 768. Cosine over the zero-padded query equals dot-product over the first N dims of the stored vector (zero positions contribute 0), giving the canonical Matryoshka semantics without re-indexing. Top-score increases monotonically with dim because more signal is included; rank is preserved. Helper: `matryoshkaTruncate(vec, dim)` in `apps/backend/convex/docsEmbed.ts`; consumed by `tools/docs/similar.ts`.

## Sandbox lifecycle (create / resume / kill)

(none yet)

## Stream pipeline (sandbox → /api/stream/event → reactive sub → client)

- **Never persist `stream_event` rows into `messages`** — Kimi partial-message mode emits per-keystroke `text_delta`s; a Vietnamese answer is 1000s of deltas. `complete` flattening each into a `messages` row blew the 2000 per-turn insert cap → "truncated: too many messages this turn" and the answer body dropped. Fix: `processBatchEvent` skips `type === 'stream_event'`; live render reads them from the `streamEvents` table (cleared after `complete`); persisted `messages` carry only canonical assistant/user/system/result/error rows. Surfaced 2026-05-16.
- **`messages.list` must be DESC + client-reverse** — was `.order('asc')` with `initialNumItems: 50`; once a turn exceeded 50 rows the final assistant block paginated off-screen → navigate away/back showed no answer. Fix: `.order('desc')`, reverse client-side in `use-chat-convex.ts`, `initialNumItems: 100`. Surfaced 2026-05-16.
- **Chat-composer uploads are `docs` rows in `mine` scope, not `/workspace` attachments** — files attached via the composer paperclip/drag run the full `docs.upload` pipeline (scan→policy→embed→materialize) and land as `docs` rows owned by the caller, materialized into the sandbox `/workspace/mine/<owner>/` mount. The composer appends a plain-language line naming them; it must NOT use "attached" / filesystem-path framing or the agent wastes turns hunting the `/workspace` root + Glob before finding them under `mine/`. Canonical tail wording points the agent at the doc tools in the `mine` scope. The dead `[FILE_ID:storageId:filename]` token shape has no resolver — never emit it; reference uploaded docs by filename (agent does `docs list --scope mine` → `docs read`). FileUpload is wired via `ChatFileUploadProvider` mounted in `default-providers.tsx`; an app with no provider shows "File upload not enabled" — every app that exposes the composer must mount it. Surfaced 2026-05-17.
- **Recreate user-visible bugs via the ACTUAL click path, not code-trace** — repeatedly this session a "code looks fine" trace missed the real defect (e.g. `getMyAttemptDetail` returned score-only for `in-progress` → take-test screen structurally unreachable for everyone). The honest reproduction is dev-sign-in (env-gated, no Google) + Playwright driving the real flow. Code-trace ≠ "seeing what the user sees." Surfaced 2026-05-17.

## Cross-user isolation

(none yet)

## Admin queue dedup

- **`listForQuarantine` merged `policyStatus='rejected'` + `scanStatus='quarantined'` with `[...a, ...b]` — a doc with BOTH states appeared twice** → React `key={r._id}` collision: `Encountered two children with the same key`. Risks duplicated or omitted rows on update. Canonical: dedupe via `Map<_id, row>` (last write wins, preserves quarantined view) before mapping to the projection. Pattern applies to any union-of-two-states list query — never trust the two slices to be disjoint.

## CLI auth (device flow + PAT)

(none yet)

## Assessment tests — generation

- **Question generation must be shared-scope-only — it was firing on user `mine` docs** — `setPolicy` + the admin scan-override path scheduled `trainingGen.generate` for ANY approved doc regardless of scope, so a user-uploaded private (`scope='mine'`) doc seeded the assessment pool — a privacy leak and spec violation (`REQUIREMENTS.md`/`question-generation-pipeline.md` = approved *shared* doc only). Fix at ≥2 points: both call-sites only schedule generation when `docs.scope === 'shared'`, AND `trainingGen.generate` itself bails `reason:'not-shared-scope'` if handed a non-shared doc (mechanism-asserted, no caller can bypass). Embedding still runs for `mine` docs (owner's own search); only generation is gated. Surfaced 2026-05-17.
- **Topic clustering by raw LLM `topicName` string fragments badly** — `persistSuggestionsWithEmbedding` created/matched topics by exact `topicName` (free-form per-question Vietnamese category) → 120 questions scattered into 80–108 micro-topics; almost none reach pool ≥5 so nothing is testable/assignable. Per `topic-clustering-plan-b.md` the canonical routing is **embedding-centroid** with a distance threshold, not name-string. Fix: cluster each question's `promptEmbedding` to nearest existing topic centroid (merge if cosine ≥ `TOPIC_MERGE_SIM`, default lowered to 0.5 for short VN MCQ embeddings), maintain running centroid, spawn new topic only when far from all. Old null-centroid topics can't absorb new questions — needs fresh regen (pre-launch wipe-sanctioned) to actually consolidate; `retireEmptyTopics` (internal) only removes truly empty husks (it found 0 — fragments aren't empty). Surfaced 2026-05-17.

## Assessment tests — review queue

(none yet)

## Assessment tests — attempts

- **`getMyAttemptDetail` must branch on status or the take-test screen is unreachable** — it returned the score-only `{score,total}` shape for ALL non-passed attempts incl. `in-progress`; the page's `'total' in attempt` branch fired → "Score 0/5 — retake" instead of the questions. Nobody could ever take a test. Canonical: `in-progress` → return `questionSnapshots` WITHOUT `correctIndexShuffled` (answer key never leaves server); terminal (passed/failed/cancelled) → `{passed, score, total, sources:[{docId,filename}]}` only. Per the founder-revised `assessment-test-overview.md`: result reveals pass/fail + score + source citations only, NO per-question answer breakdown for either outcome (more anti-cheat-safe than the old pass-reveal). Surfaced 2026-05-17.
- **Silent catch in a user action = invisible failure** — `/training` `onStart` caught errors with only `console.error` → non-tech user clicks Start, nothing happens, no feedback. Every user-facing handler must surface `toast.error`; on `not authenticated` route to sign-in. Audit other handlers for the same swallow.

## Assessment tests — assignments + re-arm

(none yet)

## Assessment tests — chat agent training tools

(none yet)

## Dashboard — top strip + cost cycle

- **`costCycleHistory` walks back fixed `30 days × i` then snaps to the 5th anchor — drifts** — months are 28–31 days, so the 30-day step accumulates error vs the 5th-of-month anchor; for many "now" dates two iterations snap into the same calendar cycle → history shows a month twice / skips one. Coincidentally correct for some dates only. Canonical (`dashboard-cost-cycle.md` = strict monthly 5th-to-5th): iterate by **calendar month** (previous month's 5th), not 30 days. Also the bar label is yearless `MM-DD` (`cycleStart.slice(5)`) → crossing a year boundary `12-05` looks suspicious; label should carry the year (`MMM YYYY`). Surfaced 2026-05-17.

## Training page

(none yet)

## costRecords aggregation

- **Only the chat-proxy path wrote `costRecords` → dashboard showed $0 despite heavy spend** — question generation, policy classifier, and `docs conflict` call Kimi via direct server `fetch`, bypassing the proxy settle that is the sole `costRecords` writer. Fix: `costRecords.recordDirect` (internal mutation; parses Kimi `usage`, prices with the SAME `computeActualCents` rate logic, upserts) called from all three direct sites — owner `'system'` for background (gen/classify), calling user for `docs conflict`. Best-effort, never blocks. Generation stays NOT budget-gated (recording ≠ gating). Verified: one gen run → dashboard `system kimi-for-coding 7205/1741 $0.05`. Surfaced 2026-05-17.
- **Rate table is fine; zero cents = zero captured tokens** — `streamHelpers` `MODEL_RATES`/`DEFAULT_RATES` are non-zero ($3/$15 per Mtok). `$0.00` means `inputTokens/outputTokens` never captured (no completed round-trip / usage not parsed), not a rate-table problem.

## Agent auto-assign cron

(none yet)

## Departments

(none yet)

## Operator-host port collisions

Operator's Colima runs many concurrent compose projects: `map_*` (timescaledb / temporal / apicurio / pgbouncer / nats / kafka / minio / valkey / glitchtip / grafana / typesense), `va_*` (web on 3002, postgres), `vbfe-*` (minio on 9000/9001, nginx on 5176, backend on 5174, postgres), `noboil_*` (convex-backend on 4100/4101, dashboard on 4102, minio on 4104/4105, spacetimedb on 4200/4103, postgres-17), `k3d-truecare-pilot-local-*` (k3d cluster). byerag ports chosen to avoid all of these: admin=3001, user=3003 (not 3002 — va-web), Convex API=3210, Convex site=3211, Ollama=11434 (compose-internal). Audit `lsof -iTCP -sTCP:LISTEN` + `docker ps --format '{{.Names}}\t{{.Ports}}'` before adding any new host-bound service.
