# assessment-test-overview

byerag includes a staff-training assessment feature. The agent reads the shared documentation corpus and generates multiple-choice questions grouped by topic. Admin curates the question pool through a review queue. Users take 5-question tests per topic; all-or-nothing pass. Admin can assign topics to all users at once; users can also voluntarily self-assess on any approved topic.

This ADR is the top-level overview; mechanism details live in the sibling ADRs listed below. Every behavior here is canonical. Future implementations must not assume or guess; if something is missing, raise as an OPEN-QUESTION and defer.

## Sibling ADRs

- [`topic-clustering-plan-b`](./topic-clustering-plan-b.md) — flat topic list; agent decides count; no hierarchy
- [`question-generation-pipeline`](./question-generation-pipeline.md) — async gen on approved doc; 10 per doc; Vietnamese; conflict scan
- [`question-review-queue`](./question-review-queue.md) — admin review actions; bulk-approve; conflict pairs; at-cap swaps
- [`question-pool-soft-cap-50`](./question-pool-soft-cap-50.md) — soft cap 50 per topic; 1-in-1-out at cap
- [`assessment-test-attempts`](./assessment-test-attempts.md) — 5 questions; 100% pass; open-book; shuffle; tab-close discard
- [`assessment-assignments`](./assessment-assignments.md) — admin assigns to role=user; real-time fire; persistent badge; un-assign nuke
- [`re-arm-on-substantive-corpus-update`](./re-arm-on-substantive-corpus-update.md) — admin flag at approval invalidates assigned passes
- [`chat-agent-training-tools`](./chat-agent-training-tools.md) — read-only own-data CLI surface

## Hard invariants

- **All AI question changes (new / revision / retire-suggested-on-contradiction) go through the admin review queue. Nothing auto-applies.** Source-doc deletion is the sole exception: cascade-delete of questions is automatic.
- **MCQ only, 3 choices, exactly 1 correct.** No short-answer, no free-form, no true/false, no multi-correct.
- **5 questions per attempt.** Random sample from the topic's admin-approved pool. Different from immediate prior attempt is best-effort only — no anti-repeat tracking.
- **100% correct = pass.** Any wrong answer = fail.
- **Unlimited retakes. No cooldown. No time limit. No deadlines.**
- **Open-book.** Every question card shows source-doc citation; doc viewer remains available beside the test.
- **Result reveals pass/fail + score + source-document citations only.** Both passed and failed attempts show the outcome (Passed/Not passed), the score ratio (e.g. `3/5`), and clickable citations of the source documents the questions were drawn from (open in the doc side-sheet). No per-question content and no correct answers are ever revealed for either outcome — the answer key never leaves the server. Citations do not leak answers (the source docs are open-book).
- **Tab close mid-attempt = discard.** No save & continue. Server marks orphan `in-progress` rows as `cancelled` when a new attempt for the same (user, topic) starts.
- **Vietnamese-only questions** regardless of source-doc language. Translation happens at generation time.
- **Question + choice order shuffled per attempt.** Anti-pattern memorization.
- **One latest attempt row per (user, topic).** Older purged on new attempt insert. Pass-state ledger (`testPasses`) keeps separate per-kind history.
- **Self-pass ≠ assignment-pass.** Distinct buckets in `testPasses`. Assignments skip already-passed-assigned users only; never auto-satisfied by prior self-passes.
- **Admin assigns to a chosen set of users or to all of them.** Targets role=user only; admins are exempt. Selective ("assign to selected") and bulk ("assign to all") both supported from the dashboard gradebook. Real-time fire via Convex reactive subscription. Each assignment is a separate persistent badge; cleared only by passing.
- **Admin un-assignment nukes all assignment rows for that topic silently.** Badges vanish; in-progress attempts cancelled; past pass records preserved.
- **Pool < 5 approved questions** blocks both user attempts and admin assignment creation. Topics with 0 approved questions hidden from the user app entirely; topics with 1-4 approved questions visible to admin but disabled on user app.
- **Substantive corpus updates re-arm assignment-passes** for affected topics. Admin flags substantive vs cosmetic at approval time. Self-passes never re-arm.
- **Chat agent reads only the caller's own training data.** No pool leak. No question content surfaced before the user passes the topic. No admin-curation access.
- **Question generation cost is ignored.** No org budget cap; per-user chat budget cap stays.

## Scope decisions baked here

- No certificates / PDF proof of completion.
- No leaderboards, no scoring beyond pass/fail.
- No manual question authoring from scratch by admin; admin curates AI-generated content (edit, force-regenerate, retire).
- No topic rename / merge / split / lock / manual-create in v0; agent-clustered flat list is canonical.
- No prerequisites between topics.
- No admin-curation gate on user-side topic visibility (every approved topic with pool ≥ 5 is self-assessable by all users).
- No assignment deadlines.
- No assignment scheduling; fires immediately.
- No anti-cheat surveillance (open-book trust posture).
- No timed mode.
- No notifications outside in-app (no email, no SMS).
- No export of questions in v0; revisit later if compliance audit demands it.
- Admin is not subject to assignments admin issues.
