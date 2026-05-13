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
- Ollama daemon running with `nomic-embed-text:v2-moe` pulled.
- ClamAV daemon.
- Host nftables ruleset applied per `network-bridge-rules.md`.
- Cron entries for daily backup + monthly restore drill.

### Added in baseline + gap-fill pass

- `compose.yml` (full stack with healthchecks + bridges).
- `apps/backend/sandbox/Dockerfile` (sandbox image source).
- Schema additions: `docs.{extractedText, lang, version, supersedes, supersededBy, deletedAt}` + nullable `docs.storageId` + `docChunks` table.
