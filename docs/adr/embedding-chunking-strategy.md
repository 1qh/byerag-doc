# embedding-chunking-strategy

Per-doc embedding is the centroid of per-chunk embeddings over the doc's `extractedText`. Chunks are computed at ingest time, embeddings computed per chunk, centroid stored on `docs.embedding`, individual chunk embeddings stored on a `docChunks` table for fine-grained similarity.

## Schema

`docChunks` table:

- `docId: Id<'docs'>`
- `seq: number` — chunk index within doc
- `start: number` — char offset start in `extractedText`
- `end: number` — char offset end (exclusive)
- `text: string` — chunk text (max ~2000 chars / ~500 tokens)
- `embedding: float64[]` — 768-dim

Indexes: `by_doc`, `by_doc_seq`. Vector index: `by_embedding` (filter fields: `docId`).

## Chunking

Sliding window over `extractedText`:

- Target chunk size: 400 tokens (~1600 chars assuming ~4 chars/token).
- Overlap: 50 tokens (~200 chars).
- Sentence-aware boundary preferred (split on `.!?\n` near the target offset); fall back to hard char boundary if no sentence break within ±100 chars.

Token counting is approximate (4 chars/token); Nomic's tokenizer is BERT-like, close enough.

## Doc-level embedding

After per-chunk embeddings computed, `docs.embedding` is the mean (centroid) over chunk embeddings. Used for `docs similar` queries that want one-hit-per-doc (the default).

## Query

`docs similar --query <text>` runs the query embedding against `docs.embedding` (doc-level centroid) by default. Returns top-K docs.

`docs similar --query <text> --granular` (P5) runs against `docChunks.embedding` and returns `(docId, chunkSeq, snippet, score)` tuples.

## Beats

- **Per-doc-single-embedding (truncate to first 512 tokens)**: loses content past the first paragraph. Semantic match on later content fails silently.
- **No chunking; one embedding per full doc via LLM long-context**: unbounded cost; doesn't fit in 512-token Nomic context anyway.
- **Per-chunk only (no doc centroid)**: forces every `docs similar` call to return chunks, not docs; harder for the agent to choose which doc to `docs read` next.

## Real cost

- One extra table (`docChunks`).
- Ingest time: O(chunks) embedding calls per doc. ~50 chunks for a 50-page PDF; <30s on CPU.
- Storage: chunk text + 768 floats per chunk. 50 chunks × 768 × 8 bytes = ~300 KB embedding per 50-page doc. Trivial.

## Gotcha for Claude

- Centroid is rough — for very long docs spanning multiple topics, the centroid can sit in "average" space far from any individual chunk. Use `--granular` for high-recall searches.
- Re-chunking on a model swap requires re-running ingest on every doc; backfill action.
- Filter fields on `docChunks.by_embedding` are limited to `docId` because `docChunks` doesn't carry `owner`/`scope` (those live on `docs`). Cross-scope similarity is enforced by first finding doc ids via vector search on `docs`, then drilling into chunks with a filtered query.
- Sentence boundary in non-English (Vietnamese, Chinese) doesn't always have `.`. Treat newlines as boundaries too; tokenizer handles the rest.
