# topic-clustering-plan-b

Topics for the assessment test are produced by Plan B: agent reads the shared corpus, emits a flat topic list, then drills into each topic to generate questions. Agent decides the topic count; admin can configure a cap (default left to agent in v0). No hierarchy, no parent-child structure, no tag overlay.

## Plan-B specifics

- Topics are agent-generated and agent-named. The label is a free-form Vietnamese-friendly string (e.g. "Bảo mật", "Chính sách HR").
- Topic list is flat — no nesting, no sub-topics, no `parentId`.
- Topic identity is a Convex `Id<'topics'>`. Topic name is mutable only by deletion + re-creation (admin manual edits deferred to v1).
- A new shared doc, after admin approval, is routed to the nearest existing topic by embedding centroid OR spawns a new topic if its embedding is farther than the new-topic threshold from all existing centroids.

## What does not happen

- No re-clustering of EXISTING approved questions on a new doc upload. Existing questions stay in their original topic regardless of how well they would fit a new topic. Per the soft-cap rule, the pool curates itself over time as 1-in-1-out swaps replace stale questions.
- No admin-initiated re-cluster button in v0.
- No tag overlay; cross-cutting concerns are not addressed by tags. A question lives in exactly one topic.
- No topic hierarchy at any level.

## Cap on topic count

In v0 the agent decides the topic count. No hard cap. No admin slider. If pathological growth is observed (e.g. >30 topics on a small corpus), revisit by adding an admin-configurable upper bound and a re-cluster mechanism in v1.

## Beats

- **Plan A (cluster after generating questions)**: produces unstable topic labels because labels are derived after-the-fact from question clusters. Outline-first (B) gives readable labels admin can trust.
- **Plan C (one topic per doc)**: loses cross-doc questions. Each topic feels like just a doc's summary, not a learning area.
- **Plan D (hierarchical tree)**: lattice-vs-tree mismatch; cross-cutting docs lose coverage; admin curation cost on rename/merge/split. v0 stays simpler.
- **Plan E (admin-defined competencies)**: front-loads admin work before the agent can generate anything. Acceptable v1 upgrade for orgs that want curriculum rigor.

## Real cost

- One LLM call to produce the topic list on first run. Subsequent doc uploads do incremental "nearest-centroid or new-topic" routing — cheap embedding math, no LLM call.
- Topic list may feel coarse on small corpora (everything ends up in 1-2 topics); admin tolerates this until corpus grows.

## MUST

- Cluster each question's `promptEmbedding` to the nearest existing topic centroid in `persistSuggestionsWithEmbedding`; merge into it if cosine ≥ `TOPIC_MERGE_SIM` (default `0.5` for short VN MCQ embeddings). Why: name-string routing scatters 120 questions into 80–108 micro-topics, none reaching pool ≥ 5.
- Maintain a running centroid per topic; spawn a new topic only when a question is far from every existing centroid. Why: embedding-centroid routing is the canonical Plan-B identity, not free-form `topicName`.
- Run a fresh regen to consolidate when topics carry a null centroid. Why: old null-centroid topics cannot absorb new questions.

## NEVER

- Create or match topics by exact free-form `topicName` string. Cost: per-question Vietnamese category fragments fracture the pool below the testable threshold.

## Pitfall

- `retireEmptyTopics` (internal only) removes truly empty husks, never fragment topics — fragments hold questions, so they are not empty and survive until a regen consolidates them.

## Gotcha for Claude

- "Agent decides count" means the first-run prompt instructs the agent to produce 5-15 topics depending on corpus size. The number is not a hard cap; it is a soft target. Document the actual prompt in `system-prompts.md` if added.
- The "new-topic threshold" — cosine distance from nearest centroid above which a new doc spawns a new topic — is a tunable constant in `constants.ts`. Default 0.5 (moderately conservative; favors joining existing topics over spawning new ones). Adjust after observing real-corpus behavior.
- Empty topics happen if all docs feeding a topic are deleted. Topic survives as an empty husk until admin deletes it via `/admin/topics`. Empty topics hidden from user app (pool = 0).
- Topic name collisions are not auto-resolved. If agent emits two topics with the same name (rare; bad LLM output), the second insert succeeds with a UUID suffix appended to disambiguate. Admin can delete one if it's redundant.
