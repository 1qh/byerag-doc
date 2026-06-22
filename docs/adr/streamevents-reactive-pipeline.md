# streamevents-reactive-pipeline

The agent inside the sandbox POSTs each SDK event to Convex `/api/stream/event`, which inserts a row in `streamEvents`. Clients subscribe to `streamEvents.by_chat_seq` for the active chat. Convex's reactive WebSocket pushes new rows to the client in order. Client buffers partial deltas (`text_delta`, `thinking_delta`, `input_json_delta`) and renders incrementally.

## Beats

- **Server-Sent Events from a custom API server**: rebuild reconnect, ordering, replay — Convex's reactive sub buys it free.
- **Direct WebSocket from sandbox to client**: client now has to authenticate to the sandbox; sandbox has client-routable port. Cross-cutting concerns explode.
- **Polling**: latency floor too high.

## Real cost

- Storage: `streamEvents` rows accumulate per chat. Mitigated by TTL on completed chats (P6+).
- Convex bandwidth: every event flows through Convex twice (sandbox POST in, reactive push out). Acceptable at internal-team scale.

## MUST

- Skip `type === 'stream_event'` in `processBatchEvent`; persist only assistant/user/system/result/error rows. Why: Kimi per-keystroke `text_delta`s are 1000s of deltas, blowing the 2000 per-turn insert cap.
- Read live render deltas from the `streamEvents` table, cleared after `complete`. Why: deltas are display-only, not durable messages.
- Order `messages.list` `.order('desc')` and reverse client-side in `use-chat-convex.ts` with `initialNumItems:100`. Why: `.order('asc')` paginates the final assistant block off-screen past 50 rows.
- Run chat-composer uploads through the full `docs.upload` pipeline (scan→policy→embed→materialize) as `docs` rows owned by the caller in `mine` scope, materialized into `/workspace/mine/<owner>/`. Why: composer files are docs, not `/workspace` attachments.
- Append a plain-language composer line naming uploaded docs, pointing the agent at `docs list --scope mine` → `docs read`. Why: "attached"/filesystem-path framing wastes agent turns hunting `/workspace` root.
- Mount `ChatFileUploadProvider` in `default-providers.tsx` in every app exposing the composer. Why: a missing provider shows "File upload not enabled".
- Reproduce a reported user-visible defect via the actual click path: env-gated dev sign-in + Playwright driving the real flow. Why: a code-trace misses defects only the real path surfaces.

## NEVER

- Persist `stream_event` rows into `messages`. Cost: 1000s of per-keystroke deltas blow the 2000 per-turn cap → "truncated: too many messages this turn", answer body drops.
- Emit the `[FILE_ID:storageId:filename]` token. Cost: dead shape with no resolver; reference uploaded docs by filename instead.
- Drive Playwright to verify a change unless the founder explicitly asks this turn. Cost: founder is the first verifier; auto-screenshotting burns time + tokens.

## Pitfall

- The reproduce-via-real-path rule covers reproducing a reported defect, not proactively verifying every change. Default flow: change → report → hand back; reach for Playwright only on an explicit "verify via playwright"/"show me" this turn.

## Gotcha for Claude

- `seq` is monotonic per chat. Agent script increments `seq` per event; client orders by `seq` not `_creationTime` (DB write order may not match emit order under retry).
- Partial deltas: client must accumulate `*_delta` events into the parent `content_block_start` block until a `content_block_stop` fires.
- `type: 'result'` event signals end of session turn; client unhooks the streaming UI state.
- Error path: agent script catches throws, POSTs a redacted `error` event (sk-ant-*, eyJ*, secret literal, IPs, `/home/user/*` scrubbed) AND THEN POSTs `/api/stream/complete` — identical termination to the success path. Posting only the error event leaves `chats.streaming === true` forever (an eternal spinner) because liveness cannot fire while retry-notice events keep arriving. The complete call flattens the error into a `messages` row, flips `streaming=false`, rotates the secret, resets the turn.
- Upstream rate-limit (HTTP 429 from the LLM) is logged distinctly as `proxy.upstream.429` (carrying `retry-after`), not lumped into the generic `non-sse-error` refund — so a throttling storm is observable and the surfaced error is meaningful. The SDK still applies its own bounded retry/backoff; on exhaustion the error-path termination above ends the chat with a clear message instead of an infinite spin.
- Liveness: action that kicked the agent schedules a `livenessCheck` action 90s later; if `streamEvents.by_chat` shows zero events at that point and chat is still `streaming`, insert an error event so the UI surfaces "agent silent" rather than spinning. Liveness only covers the zero-events case; the error-path complete above covers the events-arrive-then-fail case.
