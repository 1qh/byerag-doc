# assessment-test-attempts

A test attempt is one user taking 5 MCQ questions on one topic. All-or-nothing: 5/5 to pass, anything less fails. Open-book throughout. Unlimited retakes with no cooldown. Tab close mid-attempt discards the attempt; new attempt starts fresh. Question and choice order shuffled per attempt. Reveal of correct answers only on pass.

## Attempt lifecycle

1. **Start**: user clicks `Start test` on a topic page. Server:
   - Verifies topic exists, not deleted, approved pool size ≥ 5.
   - Cancels any prior `in-progress` attempt for the same (user, topic) by flipping its status to `cancelled` (per [`testAttempts` schema](../SCHEMAS.md)).
   - Picks 5 random questions from the approved pool. No anti-repeat tracking — best-effort uniqueness across attempts is achieved by randomness alone.
   - Shuffles the 5 picked questions into a random order for display.
   - For each question, shuffles its 3 choices into a random order; computes the new `correctIndex` for the shuffled order.
   - Pins the snapshot into the new `testAttempts` row: array of `{questionId, revision, promptText, choicesShuffled: string[3], correctIndexShuffled: number, sourceDocIds: Id<'docs'>[]}`.
   - Sets `status='in-progress'`, `startedAt=now()`, `kind='self'|'assigned'` (auto-detected: if user has a pending assignment for this topic with no passed-assigned record, kind is `'assigned'`; else `'self'`).
   - Inserts the row; returns the row id to the client.

2. **Answer**: client renders questions one-by-one or all-at-once (UX deferred). User picks a choice index per question. Server records the per-question `userAnswerIndex` into the pinned snapshot via a mutation. No partial-submit gating; client can submit each answer immediately or batch at end.

3. **Submit**: client calls `submitAttempt({attemptId})`. Server:
   - Verifies attempt row is `in-progress` and belongs to caller.
   - For each pinned question, compares `userAnswerIndex` vs `correctIndexShuffled`.
   - Sets `score = count_correct`, `finishedAt = now()`, `durationMs = finishedAt - startedAt`.
   - If `score === 5`: `status='passed'`. Insert/update `testPasses` row for `(userId, topicId, kind)` with `passedAt=now()`, `attemptId=this row`.
   - If `score < 5`: `status='failed'`. No `testPasses` write.
   - Returns the attempt row.

4. **Reveal**: client requests `attempt-detail`. If status `'in-progress'`: server returns `questionSnapshots` with `promptText` + `choicesShuffled` only (no `correctIndexShuffled`) so the user can answer. If terminal (`'passed'`/`'failed'`/`'cancelled'`): server returns `{passed, score, total, sources:[{docId,filename}]}` — pass/fail, score ratio, and the distinct source-document citations. No `questionSnapshots` and no `correctIndexShuffled` are ever sent for terminal attempts; the answer key never leaves the server, for either outcome.

5. **Retake**: user clicks `Start test` again. Cycle repeats. No cooldown. The previous `failed` attempt's row is purged on insert of the new attempt row (one-row-per-(user, topic) rule, see below).

## One-row-per-(user, topic) rule

`testAttempts` keeps exactly one row per `(userId, topicId)` at any time. On new attempt insert, the prior row for the same `(userId, topicId)` is atomically deleted (regardless of its status). Effect:

- After Alice's 7th attempt finishes (passing), attempt rows 1-6 are gone. Only row 7 (the passing one) remains.
- After Alice starts an 8th attempt (curiosity, self-kind), row 7 is gone. Only row 8 (in-progress) remains.
- Pass-state survives the row purge because `testPasses` is a separate table tracking pass records by `(userId, topicId, kind)` — Alice's passed-assigned status from row 7 persists even after row 7 is deleted.

`testPasses` is never auto-deleted by attempt lifecycle. Only invalidated by re-arm-on-substantive-update (see [`re-arm-on-substantive-corpus-update.md`](./re-arm-on-substantive-corpus-update.md)).

## Tab close mid-attempt

Server has no awareness of client-side tab close. The orphan `in-progress` row persists until either:

- User starts a new attempt for the same (user, topic): old row flips to `cancelled` atomically before new row inserts.
- Topic is deleted: pending attempts in that topic flip to `cancelled` via cascade.
- Admin un-assigns and the in-progress was an assigned-kind: row flips to `cancelled` via cascade.

No automatic timeout. `in-progress` rows can sit indefinitely if user never returns AND never starts another attempt. Acceptable: row size is small; one stale row per (user, topic).

## Anti-cheat measures

- Question + choice order shuffled per attempt: prevents users from memorizing positional patterns across retakes.
- Random sample per attempt: pool > 5 means retake shows different questions probabilistically (not guaranteed; no anti-repeat tracking).
- No correct-answer reveal on fail: user can't simply note the right answer and retake.
- All open-book: doc viewer beside test is intended; no attempt to prevent off-tab lookup.
- Single-active-tab gating (per [`concurrency-and-active-context-token.md`](./concurrency-and-active-context-token.md)) means only one tab can drive test attempts per user at a time.

## Admin actions on canonical questions

Independent of the review queue:

- **Edit** any approved question's prompt/choices/correctIndex via `/admin/topics/<id>`. Bumps `revision`. Logged at severity=low.
- **Retire** any approved question. Flips `deletedAt`; pool shrinks. Logged at severity=medium.
- **Manual create from scratch**: not supported in v0.

Edits to canonical questions do NOT retroactively update pinned attempt rows. Past attempts continue to show the version they were taken under.

## Pool-size gates

- Topic with pool = 0 approved questions: hidden from user app's training page entirely. Admin sees in `/admin/topics` with banner "no questions; testing disabled."
- Topic with 0 < pool < 5: visible on user training page with `Start` disabled. Tooltip explains "needs 5 approved questions before testable." Admin sees the same gating banner.
- Topic with pool ≥ 5: testable. `Start` enabled. Random 5 selected per attempt.

## Schema

See `SCHEMAS.md` for `testAttempts` and `testPasses` table definitions. Pin-data fields and indexes documented there.

## MUST

- Branch `getMyAttemptDetail` on status: `in-progress` returns `questionSnapshots` WITHOUT `correctIndexShuffled`; terminal (`passed`/`failed`/`cancelled`) returns `{passed, score, total, sources:[{docId,filename}]}` only. Why: a score-only shape for `in-progress` makes the take-test screen structurally unreachable.
- Surface `toast.error` from every user-facing handler; on `not authenticated` route to sign-in. Why: a silent `console.error`-only catch in `/training` `onStart` leaves a non-tech user clicking Start with no feedback.

## NEVER

- Return the score-only `{score,total}` shape for an `in-progress` attempt. Cost: the page's `'total' in attempt` branch fires "Score 0/5 — retake" instead of the questions; nobody can take a test.
- Send `correctIndexShuffled` or any per-question answer key for a terminal attempt. Cost: reveals the answer key; result shows pass/fail + score + source citations only.

## Gotcha for Claude

- The pinned snapshot in `testAttempts.questionSnapshots` is what gets shown on reveal. Don't render canonical question text on reveal — text could have changed. Reveal pulls from `questionSnapshots` only.
- `kind` on attempt is determined at start time based on the user's then-current assignment state. If admin assigns mid-attempt, kind doesn't flip; the in-progress attempt finishes as `'self'`. The newly-fired assignment requires a fresh attempt.
- `correctIndexShuffled` is the shuffled order's correct-index, NOT the canonical correctIndex. Grade comparison uses the shuffled value; canonical correctIndex is in `testQuestions.correctIndex` and is not referenced during grading.
- `revision` is captured at start time. If admin edits the canonical question while attempt is in progress, the snapshot keeps the old revision. Reveal still shows the old text via `promptText` field on snapshot.
- One-row-per-(user, topic) rule is enforced at mutation level, not by a unique index — Convex doesn't support partial unique indexes natively. Implementation must atomically delete prior row before inserting new.
- `failed` and `cancelled` attempts also overwrite prior row. So `testAttempts` has at most one row per (user, topic) regardless of attempt outcome. Past failure history lives in `auditLogs` (90-day retention) only.
- `testPasses` is the durable pass-state. Never delete unless re-arm fires for substantive corpus update (per `re-arm-on-substantive-corpus-update.md`).
- Bug risk: server must verify attempt belongs to caller on `submitAttempt`; mismatch = 403. Otherwise one user could submit another user's attempt.
