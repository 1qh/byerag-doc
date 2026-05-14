# sandbox-image-and-cli-delivery

Sandbox container runs a small Debian-based image (`node:20-slim`) with bun, agent-sdk-supporting tools (`ripgrep`, `poppler-utils`, `pandoc`, `tesseract-ocr` w/ eng+vie data, `jq`, `file`, `libmagic1`), and a non-root `agent` user. Image is built locally from `apps/backend/sandbox/Dockerfile`.

CLI delivery happens at agent-launch time, not in the image. On every `agent.run`, the Convex action writes:

- the embedded agent script (`AGENT_SCRIPT` const) to `/home/agent/run.ts`
- the embedded CLI script (`CLI_SCRIPT` const, codegen output of `packages/cli/bin/x-codegen.ts`) to `/home/agent/cli.mjs`
- per-app `SKILL.md` files under `/workspace/.claude/skills/<name>/SKILL.md`

For each provider declared in the app's `cliProviders` list, a POSIX wrapper script lands at `/home/agent/.bun/bin/<provider-name>` with content:

```sh
#!/bin/sh
exec node /home/agent/cli.mjs "$@"
```

`/home/agent/.bun/bin/` is part of the container's default `PATH` (`/home/agent/.bun/bin:/usr/local/bin:/usr/bin:/bin`), so the agent's Bash subshell resolves `<provider> <verb>` by name. Provider names are generic (`docs`, `training`); no project-repository branding surfaces in the agent-facing CLI.

The CLI dispatcher (`packages/cli/bin/x.ts`) reads its provider/command path from positional argv. When invoked via wrapper, `process.argv[1]` is the cli.mjs path; the dispatcher matches `argv[0]` against the manifest tree's provider names and walks the command path from there. The wrapper carries no name-based dispatch — provider routing is data-driven from the registry.

The CLI is a thin client: parses argv, resolves auth from `CLI_SESSION_ID` + `CLI_SESSION_SECRET` env (set by the agent-launch action), POSTs to Convex `/api/cli/exec`. Internal hosts (`localhost`, `127.0.0.1`, `convex-backend`, `host.docker.internal`) bypass the HTTPS-required check; external hosts must use HTTPS.

## Beats

- **CLI baked into image**: image rebuild required on every tool registry change. Slows iteration; image grows.
- **CLI fetched from Convex at boot**: extra HTTP at sandbox start; failure mode = no CLI on bad network.
- **CLI as separate npm package, installed in image**: publishing overhead with one consumer; circular dep with codegen.
- **Symlink at `/usr/local/bin/<provider>`**: requires root; the sandbox runs as user `agent` with `--cap-drop ALL --security-opt no-new-privileges`. Wrapper script in `/home/agent/.bun/bin/` is on container default PATH and writable by the `agent` user — the no-elevation route.
- **Single umbrella binary with sub-providers**: adds a dispatch layer the substrate's provider-as-binary mechanism does not need. Provider-as-binary keeps the dispatcher's data shape one level flat.

## Real cost

- One write of `cli.mjs` (~14 KB) + one wrapper write per provider (~45 bytes each) per agent.run. Negligible.
- Image rebuild only on toolchain change (bun version, OS deps), not on registry change.

## Gotcha for Claude

- The image's `agent` user has uid 1000; bind-mounts from host must match or use chown'd volumes.
- `node:20-slim` preinstalls a `node` user at uid 1000; the Dockerfile drops it (`userdel -r node`) before creating the `agent` user to avoid `useradd: UID 1000 is not unique`. Captured in `GOTCHAS.md`.
- `setsid bun run /home/agent/run.ts` runs the agent script in its own process group; PGID written to `/home/agent/.claude-sessions/<chatId>/agent.pgid` so kill-on-cleanup can reap children.
- `CLAUDE_CODE_INCLUDE_PARTIAL_MESSAGES=1` baked into image's `ENV` so SDK emits fine-grained deltas.
- Tesseract data packs (`tesseract-ocr-eng`, `tesseract-ocr-vie`) cover English + Vietnamese; add more on first multi-lang doc surfaces a miss.
- Image stays under ~700 MB; verify with `docker images byerag-sandbox` after build (image tag is operator-internal — not agent-visible CLI naming).
- The Claude SDK Bash tool inherits container PATH, not the parent agent process's env-override PATH; this is why the wrapper must live at a path already on the container default PATH (`/home/agent/.bun/bin/`), not at a path the parent agent adds via env (`SANDBOX_PATH`).
- Writing a wrapper through a pre-existing symlink to the cli.mjs corrupts cli.mjs (the `>` redirect follows the symlink). The agent-launch action removes the wrapper file (`rm -f`) before writing, breaking any prior symlink chain. Captured in `GOTCHAS.md`.
- The cli.mjs HTTPS-required check applies only to external hosts. `localhost`, `127.0.0.1`, `convex-backend`, `host.docker.internal` are recognized as internal and proceed over HTTP. Production deployments use HTTPS for the Convex site URL; the relaxation is for local-dev / in-container traffic only.
