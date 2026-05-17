# question-review-queue

`/admin/test-questions/pending` is the single inbox for every AI-suggested question change. Admin actions are exhaustive: Approve / Edit / Reject for new questions; Approve / Reject for retire-suggestions. Bulk-approve via checkboxes is the only bulk action in v0. Conflict pairs and at-cap swaps render as grouped cards that resolve together.

## Item kinds

- `kind: 'new'` — agent generated a fresh candidate from an approved doc.
- `kind: 'retire'` — agent suggests retiring an existing approved question because it contradicts a newly-generated one, or because the source doc was updated and the question is now stale.

Items of kind `'new'` and `'retire'` linked via `pairedWith: Id<'testQuestionSuggestions'>?` render as a single conflict pair card.

Items where the topic is at soft cap also render as a swap card pairing the new candidate with an agent-chosen retire-suggestion. Same `pairedWith` linkage.

## Per-item actions

- `'new'`: **Approve** / **Edit** / **Reject**.
- `'retire'`: **Approve** / **Reject**.

Approve persists the change to canonical `testQuestions` table; suggestion row archived with `resolvedAt`, `resolvedBy`, `resolvedAction='approve'`.

Edit opens an inline form pre-filled with the suggested content; admin tweaks; saving persists the edited version as canonical.

Reject discards the suggestion; `resolvedAction='reject'` audited.

## Bulk-approve

Each suggestion card has a checkbox. Admin can select multiple cards across the queue and click "Approve selected". Confirms in a modal: "Approve N items?" Server applies each in turn; partial failure (e.g. one item already resolved by another admin) skipped with banner notice.

Bulk-reject, select-all-in-group: deferred to v1.

## Conflict pair card

Renders both items side-by-side. Resolution options on the pair:

- **Accept swap**: approves the new question AND approves the retire of the old question atomically. One click resolves both.
- **Keep old, reject new**: rejects the new question; the retire-suggestion is also dropped (no orphan retire without a triggering new).
- **Keep both** (stretch cap): approves the new question; rejects the retire-suggestion. Pool grows past soft cap; banner warns "Pool will exceed cap. Approving will push pool to N+1."
- **Reject both**: drops the new question and the retire suggestion.
- **Edit either side individually**: same edit affordance as solo cards; the pair remains linked after edit.

## At-cap swap card

Shown when topic's approved pool is at or above soft cap and a new suggestion arrives. Agent pairs the new candidate with the most-similar-by-embedding existing approved question as a retire-suggestion. Same resolution options as conflict pair.

## Audit

Each resolution writes one row to `auditLogs`:

- `command`: one of `training.questionSuggestion.approve` / `.edit` / `.reject` / `.swapApprove`.
- `args`: JSON `{suggestionId, topicId, questionId?}`.
- `severity`: `'low'` for new approvals; `'medium'` for retire-approvals (loses content).
- `mode`: `'admin'`.

## Edge cases

- Suggestion's source doc gets deleted before admin acts → suggestion auto-rejected with `resolvedAction='auto-rejected'`, `reason='source-doc-deleted'`. Cascade-delete of canonical questions (per existing rule) handles the live pool; the pending suggestion goes away alongside.
- Topic gets deleted before admin acts → all pending suggestions for that topic auto-rejected with `reason='topic-deleted'`.
- Two admins act on the same suggestion concurrently → second action sees `resolvedAt!=null` and gets a 409; UI surfaces "another admin resolved this; refresh." No mid-air collision risk because resolution is a single mutation.

## Schema interactions

- Reads/writes `testQuestionSuggestions`.
- Approve writes to `testQuestions` (new row, or revision-bump existing).
- Approve of `'retire'` flips existing `testQuestions.deletedAt`.
- Linked via `pairedWith` for conflict and swap cards.

## Gotcha for Claude

- Edit preserves the suggestion's `sourceDocIds` and `kind`; only `prompt`, `choices`, `correctIndex` are admin-editable in the inline form.
- Approve on a `'retire'` kind: existing approved question's `deletedAt` set; pool shrinks. Per soft-cap rule this happens atomically with the paired new-question approval in a swap card.
- Source-doc deletion cascade is automatic and does NOT enter this queue. Admin already decided by deleting the doc. Only conflict-driven retires enter the queue.
