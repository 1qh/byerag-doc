# concurrency-and-active-context-token

A single user may have multiple tabs / devices open. To prevent agent-loop races on the same chat (two tabs both kicking `agent.run`), each user has one `userContexts.activeContextToken` (UUID). The active tab claims the token; other tabs render read-only "another tab is active" banner.

## Mechanism

On chat open (any tab):

1. Tab generates a random UUID, calls `users.claim({token})`.
2. Convex mutation sets `userContexts.activeContextToken = token`, `activeContextHeartbeatAt = now`.
3. Tab subscribes to `userContexts.by_user`. On token mismatch → render read-only banner.
4. Active tab heartbeats every 5s via `users.heartbeat({token})` (updates `activeContextHeartbeatAt`).
5. Stale heartbeat (>15s) makes any other tab eligible to claim.

## Send-message gating

`messages.send` mutation checks `args.token === userContexts.activeContextToken` for the user. Mismatch → 403 (`not active context`). The agent.run scheduler kick happens only on success.

## Multi-chat per user

`userContexts` is per-user, not per-chat. Active-token-per-user means only one chat-driving tab at a time per user. Switching chats in the same tab is fine (same token); switching tabs requires re-claim.

## Beats

- **No concurrency control**: two tabs kick agent.run → sandbox state race, double-billed.
- **Per-chat lock**: doesn't prevent same-user from concurrent runs across different chats.
- **Stateless (just retry)**: idempotency at agent loop is hard; cleaner to gate at the boundary.

## Real cost

- One mutation per 5s heartbeat per active user. Cheap.
- Multi-device users have to switch back to the active tab to drive — minor UX friction.

## Gotcha for Claude

- Heartbeat interval (5s) + stale threshold (15s) tuned for human tab-switch latency. Adjust if false-positive "another tab is active" surfaces.
- Banner has an "I'm active" button → forces a claim on this tab; other tabs flip to banner. Concession to power users with many tabs.
- Heartbeat over the Convex reactive WebSocket; if the WS drops, the heartbeat misses, stale fires, banner shows — operator sees "tab disconnected" implicitly.
- `userContexts.busyChatId` + `busyKind` + `busyUntil` are separate from active-token; they gate heavy pipelines (e.g., bulk embed) from concurrent kick. Active-token is the read/write gate; busy fields are the long-op gate.
