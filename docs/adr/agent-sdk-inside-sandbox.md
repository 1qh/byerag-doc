# agent-sdk-inside-sandbox

`@anthropic-ai/claude-agent-sdk` runs inside the per-owner sandbox container, not on the Convex host. SDK reads `ANTHROPIC_AUTH_TOKEN` (the per-chat proxy bearer) and POSTs to the Convex proxy as its base URL.

## Beats

- **SDK on the host (Convex action calling the model directly)**: tool execution still needs a sandbox boundary; you'd run the SDK twice (orchestrator + tool exec) or smuggle the model loop across the boundary. Cleaner to put the whole loop in the sandbox.
- **SDK on the web client**: client gets the LLM key. Never.

## Real cost

- Sandbox container needs Node + bun + SDK installed in the image. ~150MB on disk.
- SDK upgrades require image rebuild + rollout.
- Tool calls bounce: agent inside sandbox → Bash → CLI binary → Convex `/api/cli/exec` → tool action → JSON back → Bash stdout → agent. Each round trip is ~50-200ms warm.

## Gotcha for Claude

- The embedded `agentScript.ts` is the source-of-truth for the agent script. It's stored as a TS string in the Convex codebase, written to the sandbox at boot, then `setsid bun run`-ed. Edits to the script require editing the embedded string, not a separate file.
- Secrets MUST be deleted from sandbox env before SDK starts. `delete process.env.ANTHROPIC_API_KEY` + `delete process.env.CHAT_SECRET` + `delete process.env.SYSTEM_PROMPT` + `delete process.env.USER_TEXT`. The agent's tool calls can `env | grep` — anything left is leaked.
- `unstable_v2_resumeSession(sessionId)` over `unstable_v2_createSession()` when prior `sessionId` exists. Fallback on resume failure.
- PGID-scoped kill on agent exit prevents zombie children: `setsid` wrapping at launch, `pgrep -g <pgid>` + `kill -9` at finally.
- `CLAUDE_CODE_INCLUDE_PARTIAL_MESSAGES=1` enables fine-grained streaming deltas; client parsing relies on it.
