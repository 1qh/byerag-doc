# streamevents-reactive-pipeline

The agent inside the sandbox POSTs each SDK event to Convex `/api/stream/event`, which inserts a row in `streamEvents`. Clients subscribe to `streamEvents.by_chat_seq` for the active chat. Convex's reactive WebSocket pushes new rows to the client in order. Client buffers partial deltas (`text_delta`, `thinking_delta`, `input_json_delta`) and renders incrementally.

## Beats

- **Server-Sent Events from a custom API server**: rebuild reconnect, ordering, replay — Convex's reactive sub buys it free.
- **Direct WebSocket from sandbox to client**: client now has to authenticate to the sandbox; sandbox has client-routable port. Cross-cutting concerns explode.
- **Polling**: latency floor too high.

## Real cost

- Storage: `streamEvents` rows accumulate per chat. Mitigated by TTL on completed chats (P6+).
- Convex bandwidth: every event flows through Convex twice (sandbox POST in, reactive push out). Acceptable at internal-team scale.

## Gotcha for Claude

- `seq` is monotonic per chat. Agent script increments `seq` per event; client orders by `seq` not `_creationTime` (DB write order may not match emit order under retry).
- Partial deltas: client must accumulate `*_delta` events into the parent `content_block_start` block until a `content_block_stop` fires.
- `type: 'result'` event signals end of session turn; client unhooks the streaming UI state.
- Error path: agent script catches throws, POSTs an `error` event with redaction (sk-ant-*, eyJ*, secret literal, IPs, `/home/user/*` paths scrubbed) before exit.
- Liveness: action that kicked the agent schedules a `livenessCheck` action 90s later; if `streamEvents.by_chat` shows zero events at that point and chat is still `streaming`, insert an error event so the UI surfaces "agent silent" rather than spinning.
