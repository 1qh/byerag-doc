# docs-read-via-workspace-file

`docs read` writes the full extracted doc text to a workspace cache file and returns a small JSON envelope pointing at that path. The agent reads bytes via the Claude Agent SDK's native `Read` tool, not by re-running `docs read`.

## Mechanism

`docs read --id <docId>` returns one JSON object on stdout:

```json
{
  "doc_id": "k7…",
  "filename": "Decree-105-2025.md",
  "scope": "shared",
  "version": 1,
  "lang": "vi",
  "total_lines": 4823,
  "byte_size": 134_512,
  "path": "/home/agent/.docs-cache/k7….md",
  "first_lines_preview": "<first 800 chars of doc, plain>"
}
```

Envelope stays ≤2000 bytes. Full doc text written to `path` (writable per-sandbox dir at `/home/agent/.docs-cache/`, owned by `agent` user, cleared on sandbox kill).

Agent's next step is `Read <path>` (SDK native tool) when it needs the body. Read tool paginates by line range natively and handles arbitrary file sizes without truncation.

`docs read --id X --lines N-M` is also accepted for the rare case the agent wants a direct slice without going through Read, but the canonical pattern is envelope-then-Read.

## Beats

- **Raw text on stdout (prior shape)**: stdout > 2000 bytes triggers Claude Agent SDK's Bash-tool persistence layer, which writes the output to its own `tool-results/<id>.txt` and returns a `<persisted-output>` envelope with a 2KB preview. Agent rightly distrusts the preview and re-runs `docs read` / `docs grep` looking for the missing parts — loop, ~30 tool calls per turn observed.
- **Pagination at 2KB pages**: legitimate but unusable — ~500 Vietnamese chars per page, 40+ pages for one decree, agent context grows linearly.
- **MCP-server tool definitions via `createSdkMcpServer`**: properly bypasses Bash stdout caps but requires rewriting the sandbox tool-delivery design end-to-end. Defer to a later ADR if the shell-CLI surface ever stops being useful for non-agent consumers.

## Real cost

- One extra file write per `docs read` (~150KB worst case, ~5KB typical). Disk cost trivial.
- Cache dir per sandbox lifecycle, cleaned on sandbox kill — no cross-chat leakage.
- Agent has to chain `docs read` → `Read <path>` instead of `docs read` → done. One extra round trip when the doc is small; offset by zero loops when the doc is big.

## Gotcha for Claude

- Cache path must be inside the sandbox's writable area. `/workspace/*` mounts are read-only per `sandbox-image-and-cli-delivery.md`; `/home/agent/.docs-cache/` is `agent`-owned and writable. Use that.
- Envelope MUST stay under 2000 bytes including preview snippet — otherwise the SDK wraps it and we're back to the loop. Cap `first_lines_preview` at ~800 chars.
- File is per-sandbox, not per-chat. Multiple chats from the same owner share `/home/agent/.docs-cache/` (per `docker-gvisor-sandbox.md` per-owner sandbox). Safe because ACL is enforced at `docs read` time before the file is written.
- Stale entries OK to leave — overwritten on next `docs read` of same docId; whole dir blown away when sandbox is killed.
- Agent's system prompt must explicitly say: "after `docs read`, use the `Read` tool on the `path` field to view the body — do NOT re-run `docs read` to retry."
- This pattern is identical to how Claude Code's own `WebFetch` and `Read` tools handle large outputs — it is the Anthropic-canonical handoff, not a workaround.
