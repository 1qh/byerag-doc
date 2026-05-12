# bun-only-toolchain

bun is the only package manager + runtime. No npm, yarn, pnpm, npx. No lockfile committed (`bun.lock` in `.gitignore`). Deps target `"latest"`.

## Beats

- **npm**: slow install; verbose lockfile diffs.
- **pnpm**: faster install than npm but no integrated test runner.
- **yarn berry**: PnP shape doesn't match bun's runtime expectations.

## Real cost

- bun is younger; rare ecosystem edges have npm-only quirks. Workaround per-case.
- `bun update` forbidden (overwrites `"latest"` with resolved versions). Always `bun clean && bun i`.

## Gotcha for Claude

- `import { X } from 'bun'` always — never `Bun.X` global (biome `noUndeclaredVariables` flags it).
- Test runner is `bun:test`: `import { test, describe, expect, beforeAll, afterAll } from 'bun:test'`.
- `bun --bun next start` runs Next under bun runtime; without `--bun`, Next falls back to Node — version pin discipline still required.
