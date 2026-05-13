# sandbox-image-and-cli-delivery

Sandbox container runs a small Debian-based image (`node:20-slim`) with bun, agent-sdk-supporting tools (`ripgrep`, `poppler-utils`, `pandoc`, `tesseract-ocr` w/ eng+vie data, `jq`, `file`, `libmagic1`), and a non-root `agent` user. Image is built locally from `apps/backend/sandbox/Dockerfile`.

The `byerag` CLI binary is delivered to the sandbox at agent-launch time, not baked into the image. On every `agent.run`, the Convex action writes:

- the embedded agent script (`AGENT_SCRIPT` const) to `/workspace/agent.ts`
- the embedded CLI script (`CLI_SCRIPT` const, codegen output of `packages/cli/bin/x-codegen.ts`) to `/workspace/byerag.mjs`
- per-app `SKILL.md` files under `/workspace/.claude/skills/<name>/SKILL.md`

`/workspace/byerag.mjs` is symlinked to `/usr/local/bin/byerag` at boot. Agent invokes via `Bash`. The CLI is a thin client: parses argv, resolves auth from `CLI_SESSION_ID` + `CLI_SESSION_SECRET` env (set by the agent-launch action), POSTs to Convex `/api/cli/exec`.

## Beats

- **CLI baked into image**: image rebuild required on every tool registry change. Slows iteration; image grows.
- **CLI fetched from Convex at boot**: extra HTTP at sandbox start; failure mode = no CLI on bad network.
- **CLI as separate npm package, installed in image**: publishing overhead with one consumer; circular dep with codegen.

## Real cost

- One write of `byerag.mjs` per agent.run (~10 KB embedded text). Negligible.
- Image rebuild only on toolchain change (bun version, OS deps), not on registry change.

## Gotcha for Claude

- The image's `agent` user has uid 1000; bind-mounts from host must match or use `:U` mount flag (Podman) or chown'd volumes (Docker).
- `setsid bun run /workspace/agent.ts` runs the agent script in its own process group; PGID written to `/workspace/agent.pgid` so kill-on-cleanup can reap children.
- `CLAUDE_CODE_INCLUDE_PARTIAL_MESSAGES=1` baked into image's `ENV` so SDK emits fine-grained deltas.
- Tesseract data packs (`tesseract-ocr-eng`, `tesseract-ocr-vie`) chosen for English + Vietnamese; add more on first multi-lang doc surfaces a miss.
- Image stays under ~700 MB; verify with `docker images byerag-sandbox` after build.
