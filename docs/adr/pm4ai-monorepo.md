# pm4ai-monorepo

pm4ai manages the monorepo: generates `CLAUDE.md` from rules, syncs `clean.sh` / `up.sh` / `bunfig.toml` / `.github/workflows/ci.yml` / `.gitignore`, syncs `readonly/ui/` from cnsync, runs lintmax via single `bun run fix` command.

## Beats

- **Hand-curated configs per project**: drift accumulates; rule updates require N PRs across N projects.
- **Nx / Turborepo orchestration without convention enforcement**: solves task graph, not lint or codegen conventions.
- **Custom in-repo Bash scripts**: every project re-invents the same maintenance loop.

## Real cost

- pm4ai-managed files cannot be hand-edited; CLAUDE.md, ci.yml, clean.sh, up.sh, readonly/ui/ regenerate on every `bunx pm4ai@latest fix`.
- Forces "latest" dep policy; no version pinning except by ADR exception.
- Lint strictness is high; some violations require code changes that look more verbose than the obvious shape.

## Gotcha for Claude

`bun run fix` is the only lint command. Never `bun run check` (CI-only, redundant after fix). Never pipe lintmax output through `head`/`tail` — empty output IS success, failure output is already agent-formatted.

`@typescript-eslint/no-unsafe-*`, `@typescript-eslint/no-explicit-any`, `@ts-ignore` / `@ts-expect-error`, `@typescript-eslint/no-non-null-assertion` — never disabled. Fix the code, never suppress.

Single quotes, no semicolons, alphabetical imports, `node:` prefix on Node built-ins, arrow functions only, exports at end of file, named exports for utils, default export for single-component `.tsx`.
