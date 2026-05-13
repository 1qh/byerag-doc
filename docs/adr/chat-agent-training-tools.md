# chat-agent-training-tools

The chat agent (both admin app and user app) has read-only access to the calling user's own training data via new CLI commands under the `training` provider. Strict ACL: caller's own data only, no cross-user access, no pool leak (question content surfaced only for the caller's own passed attempts). Admin-curation surface (review queue, pending suggestions) is invisible to chat tools.

## CLI surface

- `byerag training status` — returns the caller's current per-topic state. Output:
  ```json
  {
    "topics": [
      {
        "topicId": "...",
        "name": "Bảo mật",
        "poolSize": 12,
        "myStatus": "passed-assigned" | "passed-self" | "in-progress" | "failed-last" | "not-attempted",
        "myLastPassedAt": "<epoch ms>",
        "myAssignmentPending": true | false
      },
      ...
    ]
  }
  ```
- `byerag training attempts` — returns the caller's recent attempt list (latest row per topic; one row per topic max per the one-row rule).
- `byerag training topics` — same as `status` but trimmed (just `topicId`, `name`, `poolSize`); useful for users asking "what topics are there?"
- `byerag training attempt-detail --id <attemptId>` — returns the full pinned snapshot of a single attempt, INCLUDING correct answers, ONLY if `attempt.status='passed' AND attempt.userId=caller`. For `failed` or `cancelled` attempts, returns `{score, total}` only — no questions, no answers.

All commands are kind-agnostic: an assigned-kind pass and a self-kind pass surface the same way in `status` (with the kind shown in `myStatus` field).

## ACL

Every action enforces caller-scope at the Convex action layer:

- `caller.email` derived from the authenticated session.
- All queries filter by `userId = caller.email`.
- `attempt-detail` additionally enforces `attempt.userId === caller.email` after row fetch; mismatch → 403.

No tool can return data for any user other than the caller. No tool can return rows from `testQuestionSuggestions`, `testAssignments.createdBy`, `auditLogs`, or any admin-only surface.

## Pool-leak prevention

The system prompt for both apps includes:

> When using `byerag training` tools, never reveal question content from any topic pool before the user has passed that topic. If the user asks "what's on the Security test?", refuse with a brief explanation that test content is private until passed.

Even attempts that the agent CAN technically access (the caller's own `attempt-detail` on a passed attempt) are gated by status: only `'passed'` attempts return full content. Failed/cancelled return scores only.

The `byerag training topics` and `byerag training status` tools never include question content; only metadata (counts, names, statuses).

The `byerag training attempts` tool returns only the metadata listed above; no `questionSnapshots`. The agent must call `attempt-detail` to get question content, which gates on `passed` status.

## What chat agent CAN do

- Tell the user their pass status per topic.
- List the user's recent attempts.
- For a passed attempt, recall the questions + the user's answers + the correct answers (reading the pinned snapshot).
- Summarize the user's training progress ("you've passed 4 of 7 active topics").
- Suggest which topic the user should take next, based on `myStatus`.

## What chat agent CANNOT do

- Show a question from a topic the user hasn't passed.
- Start a test on the user's behalf, submit answers, or simulate an attempt.
- See other users' pass status or attempt history.
- Access the admin review queue, pending suggestions, retire-suggestions, or any admin curation state.
- Modify any test data (read-only tools only).
- See deleted-topic or deleted-question content beyond what's already pinned in the caller's own past attempts.

## Schema

Tool definitions live in `apps/backend/convex/tools/training/`. Registry codegen via `packages/cli/bin/x-codegen.ts` includes them in `tools/generated/registry.ts`.

## Beats

- **No training access from chat**: forces user to switch UI surfaces (`/training` page) to check status; chat-as-natural-language is the right UX.
- **Full agent access (including admin curation)**: leaks pool content + admin-only state to user chat surface. Hard no.
- **Write access via chat (start/submit tests)**: invites cheating-via-chat ("agent, please mark me as passed"). Hard no.

## Real cost

- One small tool registry per chat session; each call is a Convex action invocation with caller-scoped queries. Cost is negligible.
- Tool surface expands the agent's prompt; mitigated by the registry-driven SKILL.md being generated on the fly per session.

## Gotcha for Claude

- The agent might be tempted to summarize the test pool ("there are questions about phishing, breach notification, ...") inferred from its knowledge of the underlying docs. This is NOT a leak through the training tools, but it IS a pool-leak via doc tools (`byerag docs grep` or `byerag docs read`). System prompt must explicitly forbid summarizing test content from doc-tool outputs as well.
- `attempt-detail` is the only tool that surfaces question content. If a passed attempt's questions were later edited (via canonical admin actions), the snapshot still shows the version the user took. This is by design — the user's view of their own training history is their own.
- `myStatus` field in `status` distinguishes `passed-assigned` vs `passed-self`. UI surface decides whether to render the distinction; agent surfaces both for completeness.
- The `byerag training` provider is enabled for both admin app and user app. Admin's chat agent can ask the same questions about admin's own training history (which is always empty in v0 because admins are exempt from assignments; admin can self-assess and that does surface).
- Future scope: a `byerag training help-me-study` tool that, after a failed attempt, points the user to specific sections of source docs without revealing the missed questions. Out of v0.
