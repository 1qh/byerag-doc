# VISION

byerag is an internal documentation assistant for a private team. Members upload documents they own; an admin uploads documents the whole team shares. Members chat with an agent that answers questions grounded in those documents, using a coding-agent driver pattern — the agent invokes filesystem-style tools (list, read, grep, diff, similar) over the corpus, composing them per query.

The agent is the driver. Retrieval-augmented generation in the classic pre-chunked-embed-and-stuff sense is not the architecture; the agent picks the tool that fits the query. Exact-match queries route to grep. Long-form reasoning routes to read + read + diff. Vague queries route to vector similarity. The model decides.

byerag self-hosts everywhere except the LLM endpoint. Documents never leave the local network as files; only document content that the agent chooses to read traverses the LLM API.

## Why this shape

A coding-agent driver replaces the brittle fixed pipeline of chunk → embed → top-k → stuff. Complex queries ("what differs between these two docs", "find every contract that mentions warranty AND ships outside EU") fall out as natural compositions of small CLI tools. Cost is bounded by per-chat turn budgets and per-owner daily caps, not by chunking strategy. Quality scales with the model's tool-use coherence, which is improving faster than retrieval pipelines.

## Non-vision

byerag is not a SaaS. byerag is not multi-tenant beyond the internal team's role split (admin / user). byerag is not a search engine — the agent answers in natural language, citing sources. byerag does not synchronize to external knowledge bases; the corpus is what users upload.
