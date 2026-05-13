# byerag autonomous build goal

Ship the byerag project end-to-end. Definition of done = all acceptance tests below pass with concrete external evidence.

## Acceptance tests (judge-verifiable)

**A. Repos public on GitHub:**
- `gh repo view 1qh/byerag --json visibility` returns `PUBLIC`.
- `gh repo view 1qh/byerag-docs --json visibility` returns `PUBLIC`.
- Both default branch `main`; HEAD on each has at least the commits referenced in B-K below.

**B. Local Docker stack on operator's Mac (Colima):**
- `docker ps --filter name=byerag` lists ≥4 services: postgres, convex-backend, ollama, clamav.
- `curl -fsS http://127.0.0.1:3210/version` returns 200 with JSON containing `version`.
- `curl -fsS http://127.0.0.1:11434/api/tags` lists model `nomic-embed-text:v2-moe`.
- Per-owner sandbox container spawns on first chat message (docker ps confirms).

**C. Web apps reachable:**
- `curl -fsS http://localhost:3001/` returns 200 (admin sign-in HTML).
- `curl -fsS http://localhost:3003/` returns 200 (user sign-in HTML).

**D. Auth end-to-end:**
- Google OAuth sign-in completes on both apps.
- `userProfiles` row for `laiquanghuy24122001@gmail.com` has `role='admin'`.
- `/admin/*` routes return 403 for role=user accounts.
- BOOTSTRAP_ADMIN_EMAIL seed works on first sign-in.

**E. Upload pipeline:**
- Clean PDF → `docs` row `scanStatus='clean', policyStatus='approved'`, blob in `_storage`.
- EICAR string upload → row `scanStatus='quarantined'`, `auditLogs` row `severity='high'`.
- Duplicate sha256 → no new row; toast surfaces "already uploaded".
- Same filename, different content → modal Replace/Keep both/Cancel; Replace produces `version=2` with `supersedes` link.

**F. Chat via Kimi proxy:**
- User sends message → SSE stream from `/api/anthropic/v1/messages` via per-chat bearer.
- Agent calls `byerag docs list/read/grep/diff/similar` tools.
- Answer includes citations linking back to docs.
- Real upstream = `api.kimi.com`.

**G. Assessment tests:**
- Approved doc triggers async gen of 10 MCQ candidates → `testQuestionSuggestions` rows status=pending.
- Admin approve → canonical `testQuestions` row.
- User passes 5/5 → `testPasses` row `kind='assigned'`.
- After `settings.agent_auto_assign_enabled='true'`, 03:00 UTC cron inserts `testAssignments` `createdBy='agent'` for eligible `(user, topic)` pairs.

**H. Dashboard at /admin/dashboard:**
- Top strip renders `<active>/<total>` users, cost cycle current `$`, docs-in-corpus count.
- Monthly cost chart + per-(user, model) pivot table for selected cycle.
- Gradebook matrix renders `✓ ✗ ⓐ ·` glyphs across role=user × non-deleted topics.

**I. Cost + audit recording:**
- `costRecords` table populated per LLM call (owner, model=`kimi-for-coding`, dayKey, tokens, cents).
- `auditLogs` has rows for every CLI exec, every bulk assign, every scan override.

**J. Test corpus + Kimi knowledge probe:**
- `apps/backend/scripts/pull-test-corpus.ts` ran on this build.
- `apps/backend/test-fixtures/probe-log.jsonl` exists with ≥5 accepted real docs (post-2026-02-01 sources) where Kimi probe returned "don't know" or hallucinated wrong.
- Edge-case fixtures present: EICAR string, prompt-injection doc, scan-only PDF, mixed-language doc.

**K. Final promise:**
- `byerag-docs/ledger.jsonl` last row's `notes` field contains verbatim string:
- `<promise>BYERAG SHIPPED — VERIFY ALL GREEN; CI GREEN; REPOS PUSHED; E2E SMOKE PASSED</promise>`

## Three valid stop reasons only

(1) Operator says stop. (2) Hard external blocker (credential/access agent cannot acquire). (3) All A-K above pass with evidence committed.

## Banned shapes

Stop: "out of scope", "diminishing returns", "deferred", "pre-existing", "good enough". Ask: "ready?", "should I?", "ok?", "plan looks good?", "what next?".

## Anti-laziness

Web-search upstream issues before declaring unfixable. Every bug = your bug. `bun run fix` before commit. Never `void mutate()`; use `.catch()`. Always `.tsx` for JSX. Never disable unsafe-* lints — fix the type.
