# ui-error-surfacing

Stream events of `type: 'error'` render inline in the chat thread as a red bordered message with the error message + a "Retry" button. Other surfaces (upload failure, sign-in failure) use toast notifications.

## Inline error (in chat)

- `messages.type = 'error'` row → renders as a `MessageError` component with red border, error icon, the redacted error text, and a "Retry" button that re-submits the previous user message.
- Persistent: stays in the chat history; doesn't dismiss.

## Toast (transient)

- Upload failure (scan rejected, oversize, network) → toast with 5s auto-dismiss + dismiss button.
- Sign-in failure (`redirect_uri_mismatch`, denied OAuth) → toast + redirect to sign-in page.
- Rate limit (`429`) → toast + 10s retry hint.

## Banner (semi-persistent)

- "Another tab is active" → banner at top of chat pane, no auto-dismiss; click "Take over" to claim.
- Network disconnect → banner; auto-dismiss on reconnect.

## Beats

- **Modal for every error**: interrupts flow; user dismisses without reading.
- **Console log only**: invisible to non-dev users.
- **All-in-one toast**: persistent errors lose context as toast disappears.

## Real cost

- Three error surfaces to maintain; each used for one shape.

## Gotcha for Claude

- Error messages from the agent already pass through the redaction layer (sk-*, JWT, IPs, paths). UI renders verbatim.
- "Retry" on a transient error re-uses the same chat session (`sessionId`); the agent picks up where it left off. On a permanent error (budget exhausted, content policy), Retry is disabled.
- Toast vs inline rule: if the user CAUSED the error (uploaded bad file, mistyped credential) → toast. If the SYSTEM produced the error (agent failed mid-turn, proxy 5xx) → inline.
