# NON-GOALS

Project-specific scope defenses. Generic engineering non-goals inherited from `book/NON-GOALS.md`.

## External SaaS for sandbox, auth, DB, queue, embeddings

Self-host everywhere. Only outbound network destination is the LLM API endpoint. No Vercel, no Auth0, no managed-Postgres, no Pinecone, no Algolia, no Sentry-as-a-service, no E2B, no Modal, no Render.

## Provider-managed compute

Sandbox runs as a local Docker + gVisor container per owner on the same machine. No Firecracker-as-a-service, no remote VM provider, no serverless container.

## Chunk-embed-stuff retrieval pipeline

The architecture is coding-agent-driver with native filesystem-style tools. Pre-chunked-and-embedded RAG is not the spec; vector similarity is one tool among many, called when the agent decides the query is semantic.

## Multi-provider AI orchestration

One LLM provider per deployment. The proxy speaks Anthropic protocol; swap upstream by changing one environment variable. No GPT + Claude + Kimi juries, no cross-provider agent committees.

## Anonymous public access

No public landing page. Google OAuth is the prod identity gate. `userProfiles.role` is the role gate (per `docs/adr/role-on-user-profile.md`). No network-segmentation requirement.

`@convex-dev/auth` Anonymous provider is registered only when `ALLOW_DEV_TOKENS=1` (env-toggle, default absence=off per `book/PHILOSOPHY.md`). Production deployments never set the flag → `providers = [Google]` end-state. Dev path exercises identical `signInImpl` + `generateTokensForSession` + `createOrUpdateUser` machinery for smoke harness coverage of session-bound rows without depending on a real Google account.

## Cross-team sharing

byerag serves one team's corpus. No marketplaces, no cross-team doc sharing, no multi-tenant. A second team gets a second deployment, not a second tenant.

## Multi-region replication

Single-instance deployment. Backups via standard Postgres dump. No multi-master, no eventual consistency primitives.

## Vector DB

Convex's native `vectorIndex` is the vector store. No Pinecone, Weaviate, Qdrant, Milvus, Chroma. Embeddings computed by a local Ollama daemon; embedding storage lives in the same Convex DB as everything else.

## Surfacing deferred items in active plans

Items filed in this `NON-GOALS.md` are filed and tracked. Agent does NOT re-list them in proactive "what's left" surfaces per `book/HARD-RULES.md` "Don't surface known-deferred items in plans".
