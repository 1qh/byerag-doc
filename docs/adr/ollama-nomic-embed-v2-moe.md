# ollama-nomic-embed-v2-moe

Embedding model: `nomic-embed-text:v2-moe` served by a local Ollama daemon. 768 dim with Matryoshka flexibility (256/512/768). Multilingual (~100 languages). MoE 8x top-2 (305M active params of 475M total).

## Beats

- **bge-small-en-v1.5 via TEI**: 384 dim, English-only, ~33M params. Cheaper but loses on multilingual corpus.
- **bge-m3 via TEI**: 1024 dim, multilingual, larger. Slower on CPU than nomic-v2 MoE for similar quality.
- **OpenAI text-embedding-3-large**: external SaaS, vendor lock-in, outbound traffic.
- **transformers.js in-process**: cold-start tax on every Node process, harder to keep model loaded.

## Real cost

- 958MB model on disk.
- ~1.5GB RAM resident while loaded.
- Ollama daemon is one more service to run (mitigated: lightweight; one binary; well-documented).
- Prefix discipline required: `search_query: <text>` for query, `search_document: <text>` for indexed doc. Skipping prefix = silent quality drop.

## Gotcha for Claude

- Pull the model once with `ollama pull nomic-embed-text:v2-moe` — model file lives in the operator-local Ollama data dir.
- API: `POST http://ollama:11434/api/embed` body `{model: "nomic-embed-text:v2-moe", input: ["search_document: " + text]}` → returns `{embeddings: [[float, ...]]}`.
- Matryoshka: store full 768 dim (per `OQ-001`). Query at 768 for top recall; truncate to 256/512 at query time if storage pressure surfaces.
- Convex `vectorIndex` cosine similarity is the default; matches nomic-embed's training objective.
- Filter fields on the vector index (`owner`, `scope`) are fixed at index creation; adding a filter is a schema migration.
- Context limit 512 tokens. Long docs need chunking before embed; chunk strategy P4 (probably sliding window 400 tokens + 50 overlap, or sentence-aware). Embedding per chunk; per-doc embedding can be the centroid or first-chunk.
