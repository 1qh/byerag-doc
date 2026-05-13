# SCHEMAS

Convex tables. Canonical. The byerag code repo's `apps/backend/convex/schema.ts` is the implementation of this spec; CI lint diffs the two per `book/HARD-RULES.md` "Spec-of-code drift bound by tooling".

## Auth

`authTables` from `@convex-dev/auth/server` — users, accounts, verification tokens. Inherited; no per-row customization.

## userProfiles

Persistent per-user metadata (separate from `userContexts` which is transient tab-active state). Per `docs/adr/role-on-user-profile.md` + `docs/adr/departments.md`.

- `userId: string` — lowercase email
- `role: 'admin' | 'user'` — access gate
- `department: 'HR' | 'Sales' | 'IT' | null` — null when role=admin or unset
- `updatedAt: number`
- `updatedBy: string` — admin email who set/changed (or `'self'` on first sign-in seed)

Index: `by_userId`, `by_role`.

## costRecords

Per-`(owner, model, dayKey)` cost + token aggregation. Drives dashboard cost analytics per `docs/adr/costrecords-table.md`.

- `owner: string` — lowercase email
- `model: string` — e.g. `kimi-for-coding`
- `dayKey: string` — `YYYY-MM-DD` UTC
- `inputTokens: number`
- `cacheCreationInputTokens: number`
- `cacheReadInputTokens: number`
- `outputTokens: number`
- `cents: number`
- `callCount: number`

Indexes: `by_owner_model_dayKey`, `by_dayKey`, `by_owner_dayKey`.

Retained forever (analytics value).

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

- `scanOverriddenBy: string?` — admin email who force-approved a scan rejection per `admin-scan-override.md`
- `scanOverriddenAt: number?`
- `scanOverrideSignature: string?` — original ClamAV signature (preserved for audit)
- `scanCancelledAt: number?` — admin pressed Cancel on a scan-override prompt
- `policyStatus: 'pending' | 'approved' | 'rejected'` — per `policy-relevance-classifier.md`
- `policyReason: string?` — classifier's short, sanitized reason
- `policyCategory: 'on-topic' | 'off-topic' | 'spam' | 'prompt-injection' | 'abusive' | 'promotional'?`
- `policyOverriddenBy: string?` — admin email when force-approved
- `policyReviewRequestedAt: number?` — when user asked admin to review

Indexes: `by_scope`, `by_owner`, `by_scope_uploadedAt`, `by_supersedes`, `by_deletedAt`, `by_sha256_scope_owner`, `by_filename_scope_owner`, `by_policyStatus`, `by_scanOverriddenBy`.

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

Every CLI exec call and every high-severity admin action.

- `owner: string`
- `command: string`
- `args: string` — JSON
- `mode: string` — auth mode the call ran under
- `ok: boolean`
- `severity: 'low' | 'medium' | 'high'?` — defaulted low; high triggers real-time alerting (P6+)

Indexes: `by_owner`, `by_command`, `by_owner_command`, `by_severity`.

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

## settings

Admin-tunable key/value strings. Known keys:

- `corpus_policy` — corpus relevance-classifier policy text. Seeded with default on first compose boot. Per `docs/adr/policy-relevance-classifier.md`.
- `agent_auto_assign_enabled` — `'true'` | `'false'`. Default `'false'`. Per `docs/adr/agent-auto-assign-cron.md`.

Schema:

- `key: string`
- `value: string`
- `updatedAt: number`
- `updatedBy: string` — admin email who last edited (or `'system'` on default seed)

Index: `by_key`.

## Assessment-test tables

Per `docs/adr/assessment-test-overview.md` and siblings.

### topics

Agent-clustered flat topic list. No hierarchy.

- `name: string` — Vietnamese label
- `autoLabeled: boolean` — true (admin manual create not in v0)
- `centroid: float64[]?` — 768-dim mean of member docs' embeddings; routes new docs
- `poolCap: number` — soft cap; default 50
- `lastSubstantiveUpdate: number?` — epoch ms; drives re-arm cascade
- `deletedAt: number?`
- `createdAt: number`

Indexes: `by_deletedAt`, `by_name`.

### testQuestions

Admin-approved canonical question pool.

- `topicId: Id<'topics'>`
- `prompt: string` — Vietnamese
- `choices: string[]` — exactly 3
- `correctIndex: number` — 0|1|2
- `sourceDocIds: Id<'docs'>[]`
- `revision: number` — starts 1; admin edit increments
- `deletedAt: number?`
- `deleteReason: 'admin-retire' | 'agent-retire-conflict' | 'source-doc-cascade' | 'topic-cascade'?`
- `createdAt: number`
- `createdBy: 'agent' | string` — `'agent'` or admin email

Indexes: `by_topic`, `by_topic_deletedAt`, `by_deletedAt`, `by_sourceDocIds`.
Vector index: `by_prompt_embedding` (`dimensions: 768`, filter fields: `topicId`, `deletedAt`) — drives duplicate scan.

### testQuestionSuggestions

Pending review-queue items + resolved-row audit trail.

- `topicId: Id<'topics'>`
- `kind: 'new' | 'revision' | 'retire'`
- `prompt: string?` — set for new/revision
- `choices: string[]?`
- `correctIndex: number?`
- `sourceDocIds: Id<'docs'>[]`
- `targetQuestionId: Id<'testQuestions'>?` — for revision/retire
- `pairKind: 'conflict' | 'cap-swap'?`
- `pairedWith: Id<'testQuestionSuggestions'>?` — reciprocal link
- `reason: string?` — agent explanation
- `hint: string?` — last regenerate hint
- `regenCount: number` — capped 5
- `status: 'pending' | 'resolved'`
- `resolvedAt: number?`
- `resolvedBy: string?`
- `resolvedAction: 'approve' | 'reject' | 'auto-rejected'?`
- `resolvedReason: 'admin-action' | 'source-doc-deleted' | 'topic-deleted'?`
- `createdAt: number`

Indexes: `by_topic_status`, `by_pair`, `by_target`, `by_resolvedAt`.

### testAttempts

One row per `(userId, topicId)` max. Older atomically deleted on new attempt insert.

- `userId: string` — lowercase email
- `topicId: Id<'topics'>`
- `kind: 'self' | 'assigned'`
- `status: 'in-progress' | 'passed' | 'failed' | 'cancelled'`
- `questionSnapshots: {questionId: Id<'testQuestions'>, revision: number, promptText: string, choicesShuffled: string[], correctIndexShuffled: number, sourceDocIds: Id<'docs'>[], userAnswerIndex: number?}[]` — exactly 5
- `score: number?` — 0-5 on submit
- `startedAt: number`
- `finishedAt: number?`
- `durationMs: number?`
- `cancelledReason: 'new-attempt-started' | 'topic-deleted' | 'assignment-cancelled'?`

Indexes: `by_user`, `by_user_topic`, `by_topic_status`, `by_status_startedAt`.

### testAssignments

Admin-issued OR agent-cron-issued; soft-deleted on un-assign.

- `userId: string`
- `topicId: Id<'topics'>`
- `createdAt: number`
- `createdBy: string` — admin email OR literal `'agent'` (per `docs/adr/agent-auto-assign-cron.md`)
- `deletedAt: number?`
- `deletedBy: string?`

Indexes: `by_user_topic`, `by_topic_deletedAt`, `by_user_deletedAt`.

### testPasses

Durable pass ledger. One row per `(userId, topicId, kind)`. Only `kind='assigned'` rows deleted by re-arm cascade.

- `userId: string`
- `topicId: Id<'topics'>`
- `kind: 'self' | 'assigned'`
- `passedAt: number`
- `attemptId: Id<'testAttempts'>` — denormalized

Indexes: `by_user_topic_kind`, `by_topic_kind_passedAt`, `by_user`.

## userContexts

Active-context-token per user, used for tab-active gating so only one tab drives heavy ops at a time.

- `userId: string`
- `activeContextToken: string?`
- `activeContextHeartbeatAt: number?`
- `busyChatId: Id<'chats'>?`
- `busyKind: 'agent' | 'pipeline'?`
- `busyUntil: number?`

Index: `by_user`.
