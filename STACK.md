# STACK

Locked stack. Each pick has an ADR with rejection rationale for alternatives. Replace by writing a new ADR amending the layer first per `book/HARD-RULES.md` "Docs-first for stack swaps and additions".

| Layer | Pick | ADR |
|---|---|---|
| Runtime + package manager | bun | `docs/adr/bun-only-toolchain.md` |
| Monorepo + lint | pm4ai + lintmax | `docs/adr/pm4ai-monorepo.md` |
| Web framework | Next.js 16 (App Router, RSC, Turbopack) | `docs/adr/nextjs-16-app-router.md` |
| Backend + DB + scheduler + auth + storage | Convex self-host | `docs/adr/convex-self-host-on-this-machine.md` |
| Auth provider | Google OAuth via `@convex-dev/auth` | `docs/adr/google-oauth-via-convex-auth.md` |
| LLM | Kimi (Anthropic-protocol-compatible) | `docs/adr/kimi-as-llm.md` |
| Agent loop | `@anthropic-ai/claude-agent-sdk` inside sandbox | `docs/adr/agent-sdk-inside-sandbox.md` |
| Sandbox runtime | Docker + gVisor (`runsc`) | `docs/adr/docker-gvisor-sandbox.md` |
| Embedding | Ollama serving `nomic-embed-text:v2-moe` | `docs/adr/ollama-nomic-embed-v2-moe.md` |
| Vector store | Convex `vectorIndex` | `docs/adr/vector-search-via-convex-vectorindex.md` |
| Upload virus scan | self-host ClamAV stateless scan service | `docs/adr/clamav-stateless-scan-service.md` |
| Frontend shared lib | `packages/react` chat hooks + components | `docs/adr/shared-react-package-for-chat-ui.md` |
| Egress | only LLM API host reachable; sandbox network-off | `docs/adr/egress-only-llm-host.md` |

## Two apps, one backend

`apps/admin` and `apps/user` are sibling Next.js apps in the monorepo. One Convex backend serves both via `chats.app` discriminator (`'admin' | 'user'`). Role is the app the user signed into, per `AUTH.md`.

## Languages

TypeScript everywhere. No Python, no Go, no Rust in this repo. pm4ai conventions enforce this via lintmax.

## Hosting

Single machine self-host. Docker compose stack on the operator's box runs Convex backend, Postgres (Convex's backing store), Ollama, ClamAV, and per-owner sandbox containers. Only outbound is the LLM API host.
