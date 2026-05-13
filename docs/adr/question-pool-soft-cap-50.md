# question-pool-soft-cap-50

Each topic's approved-question pool has a soft cap of 50. New suggestions still enter the review queue when the topic is at cap; agent pairs each new candidate with a retire-suggestion for the most-conflicting existing approved question. Admin reviews the pair; approval is a 1-in-1-out atomic swap. Admin can choose to stretch the cap by approving the new without retiring.

## Mechanism

At suggestion generation time (`question-generation-pipeline.md` stage 4), the pipeline checks the routed topic's current approved-pool size:

- `approvedCount < 50`: new suggestion enters queue solo (or paired with a retire if contradiction-scan flagged one).
- `approvedCount >= 50`: pipeline forces a pairing. For each new candidate, agent picks the most-similar-by-embedding existing approved question as a retire-suggestion. Both enter queue with `pairedWith` linkage and `pairKind: 'cap-swap'` marker (distinguishes from `'conflict'` pairs).

Admin actions on cap-swap cards mirror conflict pairs (see `question-review-queue.md`):

- **Accept swap** (default): new approved + paired existing retired atomically. Pool stays at 50.
- **Stretch**: new approved + retire rejected. Pool grows to 51. Banner warns; admin acknowledges by clicking Stretch.
- **Reject both**: new and retire both dropped. Pool stays at 50 with no change.

## Why soft, not hard

Hard cap silently skips new content when pool is full — admin doesn't see what they're missing. Soft cap surfaces every new candidate, lets admin decide whether to keep old, accept new, or stretch.

## Stretch usage

Admin can stretch arbitrarily. Pool can grow to 60 or 70 over time. Banner persists in `/admin/topics/<id>` when pool > cap: "Pool at 63, exceeds cap of 50." Admin can manually retire questions from canonical pool (see `assessment-test-attempts.md` admin actions) to bring it back to cap.

Default cap is 50. Configurable per topic via `topics.poolCap: number` (default 50). Admin can raise/lower per topic in admin UI (mechanism deferred to impl; spec carries the field).

## Choosing the retire candidate

When generating a cap-swap pair, agent's retire candidate is selected as follows:

1. Compute cosine similarity between the new question's prompt embedding and every approved question's prompt embedding in the routed topic.
2. The retire candidate is the top-1 most-similar.
3. If the top-1 is also being independently flagged as a contradiction by the contradiction scan (same iteration), use that contradiction's retire-suggestion instead of an additional cap-swap pair — avoid double-pairing.

Tie-breaking on equal cosine: prefer retiring the older question (older `createdAt`).

## Beats

- **Hard cap** (drop new content silently when full): admin doesn't see suppressed content; pool stagnates.
- **No cap** (unbounded growth): admin loses curation hold; pool drifts; retake feel changes as pool grows from 30 to 300.
- **Per-question retirement TTL** (auto-retire after N months): blind heuristic; admin has no signal for "this older question is actually still gold."

## Real cost

- Cap-swap pairing adds one nearest-neighbor lookup per new candidate. Cheap (already running embedding compare for dup-scan).
- Admin click-cost is 1 click per cap-swap card (same as solo card).
- Stretch operations accumulate technical debt: pools above cap can grow unboundedly if admin always stretches.

## Gotcha for Claude

- `pairedWith` is reciprocal: both items in the pair carry the other's id in `pairedWith`. Read/write must keep both sides consistent.
- Atomic swap on Approve: both `testQuestionSuggestions` rows updated, plus the canonical `testQuestions` insert/retire, in a single Convex mutation. Partial commit (e.g. retire applied but new not inserted) is not allowed.
- The cap is per-topic, not per-corpus. A new topic starts at pool 0 and grows freely until 50.
- Topics with `poolCap=0` reject all new suggestions silently except via stretch — useful for a topic admin has decided is "frozen." (Manual lock-style topic deferred per `topic-clustering-plan-b.md`, but the field can be repurposed for that v1 feature.)
- Soft-cap check is only at suggestion-pipeline time; it does NOT prevent admin from approving a queued non-paired solo suggestion that would push pool past cap. That's by design — admin always has final say.
