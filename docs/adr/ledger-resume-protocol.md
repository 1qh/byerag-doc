# ledger-resume-protocol

`ledger.jsonl` is the append-only outcome log at the repo root, gitignored. Each row records what just landed and what's next. The `resume` word from the founder is the recovery trigger.

## Beats

- **Plain TODO.md**: free-form; agent re-derives state from prose; loses against compaction.
- **Trello / Linear**: external SaaS; violates self-host invariant for project state.
- **Git commit messages as the ledger**: messages are unstructured, hard to query programmatically.

## Real cost

- One more file to maintain. Mitigated: append-only; one row per checkpoint; no edits.
- Gitignored, so it doesn't replicate across operator machines automatically — operator backups must include it.

## Gotcha for Claude

- Append a row after every meaningful checkpoint (commit, decision, audit conclusion, gotcha discovery, build green/red).
- The LAST row's `next` field is the resume point. `tail -1 ledger.jsonl | jq` is the read pattern.
- If the last row's `commit` SHA mismatches current HEAD: ledger is stale (work happened outside the agent loop). Re-verify state by reading current schema + tools, append a fresh row with current commit before proceeding.
- `resume` (case-insensitive, any punctuation) typed by the founder = drop in-flight, restart session-start protocol, execute `ledger.next`. No "should I?" / "ready?".
- Banned: status-summary rows in the ledger ("worked on X today"). Every row carries a `completed` (what just landed, concrete artifact) and a `next` (the next concrete action). Vague entries fail their own purpose.
- Notes field is for gotchas captured during the work. Empty if none.
