# coding-agent-as-driver-not-rag

The agent is the driver. It invokes filesystem-style CLI tools (`list`, `read`, `grep`, `diff`, `similar`) over the corpus, composing them per query. Classic RAG (fixed chunk → embed → top-k → stuff context) is not the architecture; vector similarity is one tool among many.

## Beats

- **Classic RAG pipeline**: brittle on complex queries ("compare these two docs", "find every contract mentioning warranty AND shipping outside EU", "what changed between drafts"). Pipeline is fixed-shape; query is variable-shape; mismatch tax compounds.
- **Wiki-precompute (LLM writes summaries into structured pages on ingest)**: write amplification on every doc add (one ingest touches many pages); contradictions hard to maintain; precomputed summary is stale the moment a related doc is added.
- **Single-tool retrieval (only vector OR only grep OR only LLM-read)**: query types are diverse; one tool is wrong for ≥30% of queries.

## Real cost

- Higher per-query token cost than precomputed pipelines on simple queries.
- Tool-call latency adds round trips (each tool call = sandbox → Convex → answer). Mitigated by per-chat sandbox reuse and Convex reactive subscriptions.
- Quality scales with the model's tool-use coherence — if the model is bad at multi-step tool composition, answers degrade. Acceptable trade for the query-shape coverage.

## Gotcha for Claude

- The agent should NEVER pre-chunk a doc on its own initiative ("let me embed this doc"). Chunking + embedding are P4 batch operations on ingest, not on query. On query, the agent picks the right tool.
- Don't add a tool that re-implements another tool's shape (e.g. don't add `docs full-text-search` if `docs grep` covers it). One tool per query class.
- Tool descriptions in `CLI-SURFACE.md` are the agent's instruction set. Bad descriptions → bad tool selection → bad answers. Keep them precise.
