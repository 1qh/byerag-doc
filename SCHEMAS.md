# SCHEMAS

Convex tables. Canonical. The byerag code repo's `apps/backend/convex/schema.ts` is the implementation of this spec; CI lint diffs the two per `book/HARD-RULES.md` "Spec-of-code drift bound by tooling".

## Auth

`authTables` from `@convex-dev/auth/server` — users, accounts, verification tokens. Inherited; no per-row customization.

## chats

Per-chat metadata. One row per chat thread.

- `app: 'admin' | 'user'` — which app this chat was started from
- `owner: string` — email
- `secretHash: string` — bcrypt-style hash of the per-chat proxy bearer secret
- `sessionId: string?` — Claude Agent SDK session id for resume across messages
- `streaming: boolean`
- `streamingStartedAt: number`
- `messageCount: number`
- `turns: number`
- `title: string`
- `isBookmarked: boolean?`
- `deletedAt: number?`
- `timeoutFunctionId: Id<'_scheduled_functions'>?`
- `updatedAt: number`

Indexes: `by_owner`, `by_owner_streaming`, `by_owner_updatedAt`, `by_streaming_startedAt`, `by_deletedAt`.

## messages

Sequenced events per chat (user, assistant, tool, system).

- `chatId: Id<'chats'>`
- `seq: number` — monotonic order within chat
- `type: 'user' | 'assistant' | 'system' | 'result' | 'agent' | 'error' | 'rate_limit_event' | 'stream_event'`
- `content: string` — JSON-shaped payload depending on type

Indexes: `by_chat`, `by_chat_type`, `by_chat_seq`.

## streamEvents

Append-only buffer of agent SDK events for reactive subscription. Client reassembles via parse + accumulate.

- `chatId: Id<'chats'>`
- `seq: number`
- `content: string` — JSON-encoded SDK event

Indexes: `by_chat`, `by_chat_seq`.

## chatRuntime

Per-chat counters live during a turn.

- `chatId: Id<'chats'>`
- `streamEventCount: number`
- `proxyCallsThisTurn: number?`

Index: `by_chat`.

## sandboxes

Per-owner persistent sandbox row. One sandbox per owner across all their chats; per-chat namespacing happens via `CLAUDE_CONFIG_DIR` inside the container.

- `owner: string`
- `sandboxId: string` — Docker container id
- `lastUsedAt: number?`

Indexes: `by_owner`, `by_lastUsedAt`.

## docs

The corpus.

- `scope: 'shared' | 'mine'`
- `owner: string?` — lowercase email of the owner when `scope='mine'`; null when `scope='shared'`
- `uploadedBy: string` — lowercase email of whoever uploaded
- `filename: string`
- `mime: string`
- `fileSize: number`
- `sha256: string`
- `storageId: Id<'_storage'>?` — Convex blob reference; nullable after hard-purge per `doc-versioning-and-deletion-cascade.md`
- `scanStatus: 'pending' | 'clean' | 'quarantined'`
- `summary: string?` — optional LLM-generated short summary
- `extractedText: string?` — raw text post-extraction (per `text-extraction-by-mime.md`)
- `lang: string?` — ISO 639-1 code or `'mixed'` per `multilingual-corpus-handling.md`
- `embedding: float64[]?` — 768-dim nomic-embed-text-v2-moe centroid of chunks
- `version: number` — 1-based; per `doc-versioning-and-deletion-cascade.md`
- `supersedes: Id<'docs'>?` — previous version in the chain
- `supersededBy: Id<'docs'>?` — next version in the chain
- `deletedAt: number?` — soft-delete tombstone
- `uploadedAt: number`

Indexes: `by_scope`, `by_owner`, `by_scope_uploadedAt`, `by_supersedes`, `by_deletedAt`, `by_sha256_scope_owner`, `by_filename_scope_owner`.

Vector index: `by_embedding` (`dimensions: 768`, filter fields: `owner`, `scope`).

## docChunks

Per-chunk text + embedding for a doc. Used for `--granular` similarity queries per `embedding-chunking-strategy.md`.

- `docId: Id<'docs'>`
- `seq: number` — chunk index within doc
- `start: number` — char offset start in `docs.extractedText`
- `end: number` — char offset end exclusive
- `text: string`
- `embedding: float64[]` — 768-dim

Indexes: `by_doc`, `by_doc_seq`.

Vector index: `by_embedding` (`dimensions: 768`, filter fields: `docId`).

## ownerSpend

Daily cost accounting per owner.

- `owner: string`
- `dayKey: string` — `YYYY-MM-DD` UTC
- `centsToday: number`
- `inflight: number?` — reserved-not-yet-settled cents during in-flight requests

Indexes: `by_owner`, `by_dayKey`.

## rateLimits

Rolling-window token bucket per owner / per chat.

- `owner: string`
- `tokens: number?`
- `timestamps: number[]?`
- `refilledAt: number?`
- `updatedAt: number?`

Indexes: `by_owner`, `by_updatedAt`.

## auditLogs

Every CLI exec call.

- `owner: string`
- `command: string`
- `args: string` — JSON
- `mode: string` — auth mode the call ran under
- `ok: boolean`

Indexes: `by_owner`, `by_command`, `by_owner_command`.

## CLI auth

### cliDeviceCodes

OAuth device-flow grants for the CLI.

- `deviceCode: string`
- `userCode: string`
- `userId: string?`
- `tokenId: Id<'cliTokens'>?`
- `expiresAt: number`
- `status: 'pending' | 'authorized' | 'denied' | 'expired'`
- `label: string?`
- `plaintextOnce: string?`

Indexes: `by_deviceCode`, `by_userCode`.

### cliTokens

Long-lived CLI personal access tokens.

- `userId: string`
- `tokenHash: string` — bcrypt-style
- `label: string`
- `source: 'device-flow' | 'pat' | 'dev'`
- `createdAt: number`
- `lastUsedAt: number?`
- `revokedAt: number?`

Indexes: `by_hash`, `by_user`.

### cliStreamEvents

CLI long-running command stream buffer (separate from agent streamEvents so CLI runs don't pollute chat threads).

- `runId: string`
- `userId: string`
- `seq: number`
- `terminal: boolean`
- `content: string`
- `expiresAt: number`

Indexes: `by_run`, `by_run_seq`, `by_expires`.

## userContexts

Active-context-token per user, used for tab-active gating so only one tab drives heavy ops at a time.

- `userId: string`
- `activeContextToken: string?`
- `activeContextHeartbeatAt: number?`
- `busyChatId: Id<'chats'>?`
- `busyKind: 'agent' | 'pipeline'?`
- `busyUntil: number?`

Index: `by_user`.
