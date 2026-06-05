# byerag autonomous build goal

Mission: ship byerag end-to-end. Two apps (admin + user) on self-host Convex stack with assessment-test + dashboard + agent-auto-assign features per spec.

## Read first (session start + every compact)

Full reads, no `limit`/`offset`:
1. `~/book/` ROOT only: `PHILOSOPHY.md` · `HARD-RULES.md` · `NON-GOALS.md` · `SUBSTRATE.md` · `CLAUDE.md`. NEVER subfolders.
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

Web-search upstream issues before declaring unfixable. Every bug found = your bug. `bun run fix` BEFORE commit. Never `void mutate()`; use `.catch()`. Always `.tsx` for JSX. `bun i` after workspace rename. Never disable unsafe-* lints — fix the type instead. `.github/workflows/ci.yml.disabled` stays `.disabled` until P10; "CI green" gate = `bun action` exits 0 locally (reproduce CI, do NOT re-enable the workflow file).

## Readonly files (do not edit)

`CLAUDE.md` (pm4ai-managed) · `.github/workflows/ci.yml.disabled` · `clean.sh` · `up.sh` · `bunfig.toml` · `.gitignore` · `readonly/ui/`.

## Test corpus

Per `docs/adr/test-corpus-source-and-kimi-knowledge-probe.md`. Pull post-2026-02-01 Vietnamese gov + tech docs. Probe each via Kimi before saving. Reject docs Kimi knows.

## Production deploy

Per `docs/adr/prod-deployment-pattern-dokploy.md`. Agent NEVER runs manual deploy. Dokploy API access read-only.

## Completion promise

When all three stop-condition (3) requirements above are met, emit verbatim in final ledger row's notes:

`<promise>BYERAG SHIPPED — VERIFY ALL GREEN; CI GREEN; REPOS PUSHED; E2E SMOKE PASSED</promise>`

---

## Judge-verifiable acceptance tests (for external evaluator)

External judge model passes byerag as shipped when ALL the following return expected evidence:

**A. Repos public on GitHub:**
- `gh repo view 1qh/byerag --json visibility` returns `PUBLIC`.
- `gh repo view 1qh/byerag-docs --json visibility` returns `PUBLIC`.
- Both default branch `main`.

**B. Local Docker stack on operator's Mac (Colima):**
- `docker ps --filter name=byerag` lists ≥4 services: postgres, convex-backend, ollama, clamav.
- `curl -fsS http://127.0.0.1:3210/version` returns 200 with JSON containing `version`.
- `curl -fsS http://127.0.0.1:11434/api/tags` lists model `nomic-embed-text-v2-moe`.
- Per-owner sandbox container spawns on first chat message.

**C. Web apps reachable:**
- `curl -fsS http://localhost:3001/` returns 200 (admin sign-in HTML).
- `curl -fsS http://localhost:3003/` returns 200 (user sign-in HTML).

**D. Auth end-to-end:**
- Google OAuth completes on both apps.
- `userProfiles` row for `laiquanghuy24122001@gmail.com` has `role='admin'`.
- `/admin/*` returns 403 for role=user accounts.
- BOOTSTRAP_ADMIN_EMAIL seed works on first sign-in.

**E. Upload pipeline:**
- Clean PDF → `docs` row `scanStatus='clean'`, `policyStatus='approved'`, blob in `_storage`.
- EICAR string upload → row `scanStatus='quarantined'`, `auditLogs` row `severity='high'`.
- Duplicate sha256 → no new row.
- Same filename + different content → version-conflict modal; Replace produces `version=2` with `supersedes` link.

**F. Chat via Kimi proxy:**
- User sends message → SSE stream from `/api/anthropic/v1/messages` via per-chat bearer.
- Agent calls `docs list/read/grep/diff/similar` tools.
- Answer includes citations.
- Upstream = `api.kimi.com`.

**G. Assessment tests:**
- Approved doc triggers async gen of 10 MCQ candidates → `testQuestionSuggestions` rows status=pending.
- Admin approve → canonical `testQuestions` row.
- User passes 5/5 → `testPasses` row `kind='assigned'`.
- After `settings.agent_auto_assign_enabled='true'`, the continuous 5-minute tick inserts eligible `testAssignments` `createdBy='agent'`; admin sees heartbeat + activity feed.

**H. Dashboard at /admin/dashboard:**
- Top strip renders `<active>/<total>` users, cost cycle current `$`, docs-in-corpus count.
- Monthly cost chart + per-(user, model) pivot table.
- Training page (`/admin/training`) renders KPI cards + Tests table + paginated/searchable Users roster; overdue via `assignment_due_days`.

**I. Cost + audit recording:**
- `costRecords` table populated per LLM call (owner, model=`kimi-for-coding`, dayKey, tokens, cents).
- `auditLogs` has rows for every CLI exec, bulk assign, scan override.

**J. Test corpus + Kimi knowledge probe:**
- `apps/backend/scripts/pull-test-corpus.ts` ran.
- `apps/backend/test-fixtures/probe-log.jsonl` exists with ≥5 accepted real docs (post-2026-02-01 sources) where Kimi probe returned "don't know" or hallucinated wrong.
- Edge-case fixtures present: EICAR, prompt-injection, scan-only PDF, mixed VN+EN doc.

**K. Final promise:**
- `byerag-docs/ledger.jsonl` last row's `notes` field contains verbatim:
- `<promise>BYERAG SHIPPED — VERIFY ALL GREEN; CI GREEN; REPOS PUSHED; E2E SMOKE PASSED</promise>`
