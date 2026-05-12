# vector-search-via-convex-vectorindex

Convex's native `vectorIndex` is the vector store. No external vector DB.

## Beats

- **Pinecone / Weaviate / Qdrant / Milvus**: external SaaS or external self-host. Violates self-host invariant or adds an ops surface.
- **pgvector on Postgres**: would require leaving Convex. Convex's vector index sits on top of the same backing Postgres anyway.
- **In-memory (numpy / hnswlib)**: lost on process restart; no ACL filter pushdown.

## Real cost

- Convex vector index quality is good for ≤O(100k) docs; degrades at larger scale (acceptable for internal team).
- Filter fields are declared at index creation; adding a new filter requires a schema migration.
- No re-ranking model; pure ANN cosine search. P4+ can add LLM re-rank if quality demands.

## Gotcha for Claude

- `ctx.vectorSearch(table, indexName, { vector, filter, limit })` is the only API; results sorted by descending `_score`.
- Filter pushdown is exact-match on declared filter fields. `q => q.eq('owner', uid).eq('scope', 'mine')` is the shape.
- Limit max is currently 256 (check current self-host version); paging beyond means re-querying with a different filter.
- Cosine similarity; vectors do NOT need to be normalized client-side (Convex handles it).
- Embedding field must be `v.array(v.float64())` of exactly `dimensions` length. Mismatched length on insert throws.
- Re-embedding (model version bump) is a batch backfill action; no automatic re-embed.
