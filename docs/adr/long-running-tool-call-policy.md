# long-running-tool-call-policy

`docs grep` and `docs similar` over large corpora can take longer than the agent's typical tool-call window. Each tool has a per-call deadline; longer ops use the streaming exec path.

## Per-call deadlines

| Tool | Soft deadline | Hard deadline |
|---|---|---|
| `docs list` | 2s | 10s |
| `docs read` | 5s | 30s |
| `docs grep` | 10s | 60s |
| `docs diff` | 10s | 60s |
| `docs similar` | 5s | 30s |
| `docs similar --granular` | 10s | 60s |

Soft deadline → tool returns a partial result with `truncated: true` + `nextCursor`. Agent can call again with `--cursor` to continue.

Hard deadline → tool aborts; returns `{error: {category: 'transient', code: 'TIMEOUT'}}`. Agent retries or surfaces failure.

## Streaming exec (for very long ops)

`byerag --stream <provider> <cmd>` invokes the streaming path. Returns NDJSON: intermediate events (`{kind: 'progress', percent: N}`) + terminal (`{kind: 'complete' | 'failed'}`).

P3+ scope: `docs similar --granular --stream` for high-K searches; `docs grep --stream` for full-corpus regex sweeps.

## Beats

- **No deadlines**: misbehaving tool hangs the agent loop; budget burns on idle wait.
- **Single global deadline**: short tools wait forever; long tools time out artificially.
- **Async-only API**: every tool call becomes two-phase (kickoff + poll); doubles round-trip cost for the common case.

## Real cost

- Per-tool deadline table to maintain.
- Pagination logic in the action layer.

## Gotcha for Claude

- `truncated: true` is the agent's signal to either keep paging or stop and answer with what it has. The system prompt instructs the latter for cost — pagination is opt-in, not default.
- Stream-exec uses `cliStreamEvents` table for event buffer; client subscribes to that table for live render.
- Hard-deadline abort uses `AbortController` against the Convex action; Convex's action runtime respects it.
- Soft-deadline check is cooperative (the tool checks `Date.now()` against its budget mid-loop). Non-cooperative loops can't be soft-deadlined; mark them `--stream`-only.
