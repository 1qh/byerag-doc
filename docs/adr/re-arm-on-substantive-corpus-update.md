# re-arm-on-substantive-corpus-update

When admin approves a batch of review-queue items for a topic that includes either contradiction-driven retires or admin-flagged-substantive new questions, the topic is marked `lastSubstantiveUpdate`. All users whose latest assigned-kind pass on that topic was timestamped before that marker have their `testPasses(kind='assigned')` row deleted, fresh assignments are re-fired. Self-passes are never affected.

## Trigger

A batch of review-queue resolutions for a single topic is committed as one logical operation in the admin UI. At commit time, the server inspects the batch:

- Contains at least one `'retire'` approval (any kind: cascade-cleanup retires, contradiction-driven retires, cap-swap retires) → batch is **substantive by default**.
- Contains only `'new'` approvals and the admin explicitly toggled `Mark batch as substantive` → substantive.
- Contains only `'new'` approvals and admin did NOT toggle → **cosmetic**.
- Contains only `'revision'` approvals → substantive by default (wording changed by an underlying doc).
- Contains only `'edit'` (admin manual edits via the canonical-pool admin actions in `/admin/topics/<id>`) → cosmetic by default; admin can toggle substantive.

The admin commit modal shows the inferred classification and the substantive toggle. Default value reflects the inference. Admin can flip either way before clicking Commit.

If substantive: server writes `topics.lastSubstantiveUpdate = now()` and triggers the re-arm cascade.

## Re-arm cascade

For the affected topic, server walks `testPasses.by_topic_kind` where `kind='assigned' AND passedAt < topics.lastSubstantiveUpdate`. For each match:

1. Delete the `testPasses` row.
2. Insert a fresh `testAssignments` row for the same `(userId, topicId)` if one doesn't already exist (idempotent — admin's prior assignment cascade may have created one already; reactive sub already shows it).
3. Audit log row per user with `command='training.assignment.rearm'`, `args={userId, topicId}`, `severity='medium'`.

Users receive the fresh badge via reactive sub.

## Self-pass invariance

`testPasses` rows where `kind='self'` are untouched by re-arm. Users who voluntarily self-assessed before the update keep their self-pass record indefinitely. Self-passes never satisfy assignments per the locked rule, so self-pass status's only purpose is the training-page badge for the user's own information.

## What counts as "substantive" — concrete examples

| Scenario | Default classification | Admin override |
|---|---|---|
| Admin approves 5 new questions for a topic, no retires | cosmetic | yes, can flip substantive |
| Admin approves a cap-swap card (1 new + 1 retire) | substantive | yes, can flip cosmetic |
| Admin approves a contradiction-pair card | substantive | yes, can flip cosmetic |
| Admin force-regenerates a question via canonical admin action | cosmetic | yes (when reviewing the resulting suggestion) |
| Admin edits a question's typo via canonical admin action | cosmetic | yes |
| Admin retires a question via canonical admin action (no new replacing) | substantive | yes |
| Source doc deleted → cascade question retire | substantive (automatic, no batch) | no override; cascade fires re-arm unconditionally |

## Cascade-from-doc-delete branch

When a source doc is hard-deleted (or its 30-day soft-delete grace ends), the question cascade fires. This is NOT subject to admin batch review; it's automatic. The cascade ALSO triggers `lastSubstantiveUpdate=now()` and the re-arm cascade for affected topics. No admin override at this point; admin's earlier doc-delete click implied the corpus change.

## When this DOES NOT re-arm

- Topic deletion: assignments for that topic are nuked entirely (per `assessment-assignments.md` cancel cascade). Pass records preserved as historical audit but become orphan (refer to a deleted topic).
- Self-pass-only changes: admin actions that affect only self-passes (none in v0 because admin doesn't have a "delete self-pass" action).
- Pure pool-grow without contradiction: admin approves 10 new questions, no retires, doesn't flag substantive. Pool grows, topic still searchable; no re-arm.

## Schema

- `topics.lastSubstantiveUpdate: number?` — epoch ms; null until first substantive update.
- `testPasses.passedAt: number` — compared against the topic's `lastSubstantiveUpdate` to determine staleness.

## Gotcha for Claude

- Re-arm is a one-shot event per substantive commit. If admin commits two substantive updates 1 hour apart, the second commit re-arms anyone who passed between the two commits as well as anyone whose pass predates both. No accumulation; the latest `lastSubstantiveUpdate` is what staleness is measured against.
- Admin's substantive flag has no DB column on the batch itself; it's just the inference + override that decides whether to write `lastSubstantiveUpdate` on commit. Audit log records `args.substantive: bool` for forensics.
- Cascade fires after the batch commits, not before. If the batch fails partway through (e.g. server error mid-write), no re-arm. Recovery: admin re-attempts the batch; idempotency holds because already-applied resolutions skip on re-attempt.
- High-frequency substantive updates can spam users with badges. Default heuristic biases cosmetic so admin must opt-in to substantive for new-only batches. Tune in admin UI/UX iteration if false-positives emerge.
