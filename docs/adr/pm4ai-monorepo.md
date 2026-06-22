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

## Auto-fix races

### MUST
- Prefer `.collect()[0]` over `.first()` when the result feeds a subsequent `.patch()`: `const rows = await ctx.db.query(...).withIndex(...).collect(); const existing = rows[0]`. Why: `bun run fix` auto-fix strips `await` from chained Convex `.first()` calls even with a typed annotation, never from `.collect()`.
- Make ALL edits first, then run `bun fix` to completion in the foreground with zero pending edits. Why: `bun fix` rewrites files in place and silently reverts edits made while it runs.
- Confirm `pgrep -f 'lintmax|bun.*fix'` is clear before editing any file. Why: an in-flight formatter races the edit and writes its pre-edit buffer back.

### NEVER
- `run_in_background` a `bun fix` and keep editing. Cost: the formatter writes the pre-edit buffer back, silently reverting the change.
- Drop `await` from a Convex `.first()` whose result feeds `.patch()`. Cost: the variable becomes a `{kind:'first',...}` query token and `.patch(existing._id, ...)` blows `Must provide arg 1 'id' to 'patch'`.

### Pitfall
- The auto-fix "remove redundant await on non-Promise" pass misclassifies a Convex thenable when the value variable looks unused; a same-expression use (e.g. `if (existing?._id)`) keeps the `await` shape.
