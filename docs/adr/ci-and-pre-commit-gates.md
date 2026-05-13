# ci-and-pre-commit-gates

pm4ai syncs `.github/workflows/ci.yml` (universal). byerag adds local pre-commit gates via `lefthook` and a small set of byerag-specific CI checks.

## Pre-commit (lefthook)

- `bun run fix` — lintmax runs all 5 linters; silent on success, fails noisy on violation.
- `bun run test` — bun test runs unit + integration tests under `apps/*/`, `packages/*/`.
- `bun run check:schema-drift` — diffs `byerag-docs/SCHEMAS.md` against `apps/backend/convex/schema.ts`; fails on mismatch per `book/HARD-RULES.md` "Spec-of-code drift bound by tooling".
- `bun run check:doc-leak` — greps for banned strings in code-repo source (`/^(exim|eximagent|TMDB|claude2b|Apollo|SalesQL|Modal|Serper|SendGrid|Vertex|Typesense)/`); fails on hit.
- `bun run check:secret-leak` — greps for shapes of secrets (`sk-ant-`, `sk-kimi-`, `eyJ`-JWT, 40-char hex, etc.) in any tracked file; fails on hit.

## CI (`.github/workflows/ci.yml`, pm4ai-managed)

- Runs pre-commit checks again (no shortcut).
- Runs `bun run smoke:agent` against a fresh local stack (Postgres + Convex + sandbox image build).
- Builds the sandbox image, tags `byerag-sandbox:ci`, runs a smoke from `tests-e2e/`.

## Schema drift check shape

```ts
const specPath = 'byerag-docs/SCHEMAS.md'
const codePath = 'apps/backend/convex/schema.ts'
const specTables = parseSpecTables(spec)   // names + field lists
const codeTables = parseCodeTables(code)   // names + field lists from defineTable()
diff(specTables, codeTables) → fails CI on mismatch
```

Lives in `byerag/scripts/check-schema-drift.ts`. Per `book/HARD-RULES.md` spec-of-code allowlist requires this lint for any ADR that names a code artifact as its SoT — `SCHEMAS.md` does.

## Doc-leak check

`scripts/check-doc-leak.ts` greps source. Allowlist of expected occurrences (e.g., the word `Modal` for the JSX modal dialog) lives in the script. CLAUDE.md exempted because pm4ai regenerates it.

## Beats

- **No pre-commit, CI-only**: feedback loop too slow.
- **CI-only schema drift**: drift lands then breaks main; pre-commit catches it earlier.
- **Hand-managed CI per project**: drifts across projects; pm4ai's universal `ci.yml` keeps the bones uniform.

## Real cost

- Pre-commit adds 10-30s per commit (lint + tests).
- Schema drift check requires keeping spec + code in sync; small discipline cost.

## Gotcha for Claude

- `bun run check` is forbidden per pm4ai; `bun run fix` is the only command. Lefthook calls `fix`, not `check`.
- Schema drift check is sensitive to ordering of fields in the spec; canonical order is alphabetical.
- CI runs in clean checkout; secrets live in CI's secret store, never committed. Schema drift + doc-leak + secret-leak checks run on every push including PRs from forks (security-pinned actions only).
