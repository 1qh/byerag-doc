# ollama-nomic-embed-v2-moe

Embedding model: `nomic-embed-text-v2-moe` (dashes, single segment slug per Ollama library at `ollama.com/library/nomic-embed-text-v2-moe`). Served by **Mac-native Ollama daemon** (NOT containerized). 768 dim with Matryoshka flexibility (256/512/768). Multilingual (~100 languages). MoE 8x top-2 (305M active params of 475M total).

## Hosting posture

Operator's macOS host runs the native Ollama daemon (`ollama serve` → `:11434`). Compose stack reaches it via `host.docker.internal:11434`. Cross-host bridge is the standard Colima / Docker Desktop affordance — no container build, no Linux-VM cold-start, no model duplication across the VM boundary.

`ollama serve` on Mac is fast (Metal-accelerated on Apple Silicon), uses unified memory efficiently, and the model file lives in the operator's `~/.ollama/` cache once pulled — never wasted on a Colima volume that gets wiped.

## API surface

Use Ollama's **OpenAI-compatible** endpoint, NOT native `/api/embed`:

- Endpoint (from Mac host): `http://127.0.0.1:11434/v1/embeddings`
- Endpoint (from Convex container): `http://host.docker.internal:11434/v1/embeddings`
- Request:
  ```
  POST /v1/embeddings
  {
    "model": "nomic-embed-text-v2-moe",
    "input": ["search_document: <text>"]
  }
  ```
- Response (OpenAI shape): `{"data":[{"embedding":[float,...]}], "model":"...", "usage":{...}}`.

OpenAI-compat shape future-proofs against swapping providers (drop-in Cohere / Voyage / etc.) without rewriting the Convex action.

## Beats

- **Containerized Ollama in compose**: wastes ~1 GB RAM in the Linux VM, duplicates the model file, slower on Apple Silicon than Metal-accelerated host build. Drop.
- **Native `/api/embed` Ollama endpoint**: tighter to Ollama but doesn't translate when swapping providers. OpenAI-compat is the portable surface.
- **bge-small-en-v1.5 via TEI**: 384 dim, English-only. Loses on multilingual corpus.
- **bge-m3 via TEI**: 1024 dim, multilingual, larger. Slower on CPU than nomic-v2 MoE for similar quality.
- **External SaaS** (OpenAI text-embedding-3-large, Cohere, etc.): outbound traffic; violates self-host-only-LLM-outbound rule.

## Real cost

- 957MB model on disk in operator's `~/.ollama/`.
- ~1.5GB RAM resident while loaded on host (shared with operator's other Ollama models).
- Daemon process `ollama serve` running on host; one less service in compose.
- Prefix discipline required: `search_query: <text>` for query, `search_document: <text>` for indexed doc. Skipping prefix = silent quality drop.

## Gotcha for Claude

- **Model slug is `nomic-embed-text-v2-moe`** — dashes, single segment. NOT the colon-tag form `nomic-embed-text:v2-moe` (colon = wrong; that's a tag of a different base model that doesn't exist). The official library URL `ollama.com/library/nomic-embed-text-v2-moe` is authoritative.
- **Ollama is NATIVE on Mac, NOT in compose.** Don't add an `ollama` service to `compose.yml`. The compose stack reaches the host daemon via `host.docker.internal:11434`.
- **Native-on-host vs in-container model state**: pulling via `docker exec` into a container would target the container's data dir, not the operator's `~/.ollama/`. Always pull via host shell: `ollama pull nomic-embed-text-v2-moe`.
- **`host.docker.internal` works on Colima** (operator's setup) and Docker Desktop. On native Linux Docker, this hostname requires `--add-host=host.docker.internal:host-gateway` (handled by compose `extra_hosts` per service). byerag prod deploy adjusts as needed.
- **OpenAI-compat endpoint vs native**: Ollama's `/v1/embeddings` is the OpenAI shape; `/api/embed` is native. Use the OpenAI shape for portability + future provider swap.
- **Daemon lifecycle**: `ollama serve` runs as a background process on the operator's host. macOS bg-daemon discipline: `pgrep -f 'ollama serve'` checks, `ollama serve >/tmp/ollama.log 2>&1 &` starts if absent. Operator may also use `brew services start ollama` for managed lifecycle.
- **Matryoshka**: store full 768 dim. Query at 768 for top recall; truncate to 256/512 at query time if storage pressure surfaces.
- **Context limit 512 tokens** for the model. Long docs need chunking before embed (per `embedding-chunking-strategy.md`).
- **Convex container reaches Mac-native Ollama** by `host.docker.internal:11434`. Convex backend's `extra_hosts: ["host.docker.internal:host-gateway"]` may need to be added to compose.yml for cross-platform parity. Verify in P4 implementation.
