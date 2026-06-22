# kimi-as-llm

LLM provider is Kimi via its Anthropic-protocol-compatible endpoint at `https://api.kimi.com/coding/`.

## Beats

- **Anthropic direct**: higher cost; Convex proxy + agent SDK already speak Anthropic protocol — Kimi is a drop-in.
- **OpenAI / Gemini direct**: different protocol; SDK swap costs.
- **Local model (vLLM serving Qwen / Llama)**: quality at internal-team scale is workable but tool-use coherence trails frontier models materially; agent-driver loop benefits from frontier reasoning. Revisit if regulatory pressure mandates air-gap.

## Real cost

- Outbound traffic to `api.kimi.com:443` (Beijing region by default). Acceptable under the project's "LLM outbound only" constraint.
- One vendor lock-in for inference (mitigated: proxy upstream is a single env var; swap to Anthropic / Bedrock-in-VPC / local model by changing the URL).
- Single model endpoint (`kimi-for-coding`); `model` param in requests is ignored by Kimi's coding endpoint.

## MUST

- Mandate a plain-text final-answer response after the tool chain via the system prompt. Why: Kimi ends turns with `stop_reason="tool_use"`, never emitting follow-up text.
- Assert non-empty assistant text + keyword + citation per scenario in the smoke harness. Why: catches the empty-result hole at the gate.

## NEVER

- Trust Kimi to emit a synthesizing text turn after the last tool call. Cost: `stop_reason="tool_use"` leaves `result.result=""`, chat shows no answer body.

## Gotcha for Claude

- Bearer / x-api-key both work on Kimi's endpoint. SDK sends `Authorization: Bearer <key>` by default after `ANTHROPIC_AUTH_TOKEN` is set.
- Prompt caching works (verified `cache_read_input_tokens` >0 after warm). Anthropic-beta header `prompt-caching-2024-07-31` accepted.
- Tool use shape exact: parallel `tool_use` blocks, multi-turn `tool_result` round trips, sys-prompt obedience all verified.
- Streaming SSE events use Anthropic event names (`message_start`, `content_block_start`, `content_block_delta`, etc.).
- Long context (20K input tokens) needle retrieval clean; haven't verified the upper bound. Document advertises 256K.
