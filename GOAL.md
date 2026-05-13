# byerag autonomous build goal

Mission: ship byerag end-to-end. Two apps (admin + user) on self-host Convex stack with assessment-test + dashboard + agent-auto-assign features per spec.

## Read first (session start + every compact)

Full reads, no `limit`/`offset`:
1. `~/tc/book/` ROOT only: `PHILOSOPHY.md` · `HARD-RULES.md` · `NON-GOALS.md` · `SUBSTRATE.md` · `CLAUDE.md`. NEVER subfolders.
2. `~/codoc/byerag-docs/` ROOT + every ADR under `docs/adr/`.
3. Agent memory `~/.claude/projects/-Users-o-codoc/memory/` MEMORY.md + every pointer file.
4. `~/codoc/byerag/` filesystem inventory.
5. `~/codoc/byerag-docs/ledger.jsonl` last row's `next` field = current action.

## Execute

Per `ROADMAP.md`: P0 → P10 in order. Single pass. No deferring within scope.

P0 substrate · P1 sandbox+proxy · P2 docs corpus · P3 agent tools · P4 embedding+similar · P5 chat UX · P6 cost+audit · P7 verify+harden (pull real test corpus + Kimi-knowledge probe FIRST) · P8 assessment tests · P9 departments+dashboard+agent-auto-assign · P10 polish.

## Validate

Every `- [ ]` in `VERIFY.md` → `- [x]` ONLY with evidence in ledger.notes (terminal output, commit SHA, log snippet, test result). Untested checks don't count.

## Commit + push discipline

After every logical unit: `bun run fix` (silent on success) → `git commit --no-verify -m '<conventional>'` → `git push origin main`. CI is `.disabled` until P10. Append ledger row per checkpoint.

## Banned shapes

Stop: "out of scope", "diminishing returns", "real fix later", "deferred", "pre-existing", "good enough".
Ask: "ready?", "should I?", "ok?", "plan looks good?", "what next?".

## Three valid stops only

(1) founder-says-stop. (2) hard external blocker (cred/access agent cannot acquire). (3) ALL VERIFY green + CI re-enabled + green + repos pushed + E2E smoke (real upload → chat → assessment → cost → gradebook) all error-free.

## Anti-laziness

Web-search upstream issues before declaring unfixable. Every bug found = your bug. `bun run fix` BEFORE commit. Never `void mutate()`; use `.catch()`. Always `.tsx` for JSX. `bun i` after workspace rename. Never disable unsafe-* lints — fix the type instead.

## Readonly files (do not edit)

`CLAUDE.md` (pm4ai-managed) · `.github/workflows/ci.yml.disabled` · `clean.sh` · `up.sh` · `bunfig.toml` · `.gitignore` · `readonly/ui/`.

## Test corpus

Per `docs/adr/test-corpus-source-and-kimi-knowledge-probe.md`. Pull post-2026-02-01 Vietnamese gov + tech docs. Probe each via Kimi before saving. Reject docs Kimi knows.

## Production deploy

Per `docs/adr/prod-deployment-pattern-dokploy.md`. Agent NEVER runs manual deploy. Dokploy API access read-only.

## Completion promise

When all three stop-condition (3) requirements above are met, emit verbatim in final ledger row's notes:

`<promise>BYERAG SHIPPED — VERIFY ALL GREEN; CI GREEN; REPOS PUSHED; E2E SMOKE PASSED</promise>`
