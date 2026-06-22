# CLI-SURFACE

Canonical agent-facing CLI commands. Each command is a Convex action registered in the tool registry; the CLI binary is a thin dispatcher.

Pattern: `<provider> <verb> [--flag value ...]`. Each provider installs as its own top-level binary in the sandbox. Provider name = binary name. P0 providers: `docs`, `training`.

## docs list

List documents in scope. Returns id, filename, mime, size, uploadedAt, scope.

- `--scope` (union) required enum: `shared | mine | both`
- `--limit` (number) — default 50, max 200

## docs read

Materialize a document's extracted text to a per-sandbox cache file and return an envelope pointing at the path. Agent reads the body via the Claude Agent SDK's native `Read` tool on the returned `path`. Per `docs/adr/docs-read-via-workspace-file.md`.

- `--id` (string) required — `docs._id`
- `--lines` (string) — optional line range `N-M`; when set, also includes a `body` field with just that slice for direct inline use (envelope stays ≤2000 bytes)

Returns one JSON object on stdout:

```json
{
  "doc_id": "k7…",
  "filename": "Decree-105-2025.md",
  "scope": "shared",
  "version": 1,
  "lang": "vi",
  "total_lines": 4823,
  "byte_size": 134512,
  "path": "/home/agent/.docs-cache/k7….md",
  "first_lines_preview": "<≤800 chars from line 1>"
}
```

Full doc text is written to `path` (sandbox-writable `/home/agent/.docs-cache/`, cleared on sandbox kill).

ACL: row is fetched, if `scope='mine'` and `owner != caller.email` → `FORBIDDEN`.

## docs grep

Regex match across docs in scope. Returns matching `(docId, lineNumber, snippet)` tuples.

- `--pattern` (string) required — regex (re2 dialect)
- `--scope` (union) required enum: `shared | mine | both`
- `--limit` (number) — default 50, max 200

Implementation: iterate `docs` rows in scope (paginated), pull extracted text from `_storage` (or cached text field), run regex, return hits.

## docs diff

Mechanical unified diff between two docs (line-level).

- `--a` (string) required — `docs._id`
- `--b` (string) required — `docs._id`
- `--context` (number) — lines of context, default 3

ACL: both rows checked individually. Returns raw diff text.

## docs conflict

Semantic conflict scan between two docs. LLM-driven; structured output. Per `docs/adr/auto-resolve-via-shared-kb-on-conflict.md`.

- `--a` (string) required — `docs._id`
- `--b` (string) required — `docs._id`

Returns JSON array `[{type: 'factual' | 'wording' | 'gap', summary, docA_excerpt, docB_excerpt}]`. Excerpts are literal substrings of source texts (grep-verified by server before return; hallucinated excerpts dropped).

ACL: both rows checked. Chunk-pair fallback when combined text > 100K chars.

Auto-probe rule baked in system prompt: for each `'factual'` conflict, agent runs `docs similar --scope shared` and `docs read` on top match (cosine ≥ 0.8) to surface canonical authority + recommend escalation if no canonical found. Hard cap 3 canonical probes per user-question.

## docs similar

Vector similarity over `docs.embedding`. Returns top-K with cosine score, filename, snippet.

- `--query` (string) required
- `--scope` (union) required enum: `shared | mine | both`
- `--limit` (number) — default 10, max 50
- `--dim` (number) enum: `256 | 512 | 768` — Matryoshka prefix dim for storage-efficient search; default 768

Implementation: query text prefixed with `search_query: `, embedded via Ollama, `ctx.vectorSearch('docs', 'by_embedding', { vector, filter, limit })`.

## docs summarize (P5)

Ask the agent to produce a fresh summary of a doc (stored back into `docs.summary`).

- `--id` (string) required
- `--style` (union) enum: `terse | bullet | abstract` — default `bullet`

## docs cite (P5)

Mint a stable citation handle from a doc id + line range, used in agent answers.

- `--id` (string) required
- `--from` (number) — line, default 1
- `--to` (number) — line, default `min(end, from+20)`

## training (P8)

Read-only own-data tools the chat agent uses to coach the user through assessment tests. Per `docs/adr/chat-agent-training-tools.md`. Provider binary: `training`.

- `training status` — caller-scoped topic-status list
- `training attempts` — caller-scoped attempt list
- `training topics` — topic list with pool sizes; no question content
- `training attemptDetail --id <attemptId>` — full snapshot on caller's passed attempt; score-only on failed/cancelled

## Output shape

All commands return JSON on success. On error: `{error: {category, code, message, retryable, details?}}` and non-zero exit. Categories: `auth | input | permanent | transient | upstream`.

## Brand

Provider binaries are generic names (`docs`, `training`) — agent-visible surface carries no project-repository-name branding. The repo name is internal-only.

## CLI builder defaults

### MUST
- Declare a defaulted arg as `arg.number({ default: 50, description: '…' })` — no `optional: true`, no `??` in handler. Why: `default:` auto-marks the arg optional and auto-substitutes at handler entry.
- Read a defaulted arg directly as `Infer<V>`, never `| undefined`. Why: `applyDefaults` runs inside `defineTool`/`defineQuery`/`defineMutation` before the user handler, so the value is always present.

### Pitfall
- Spec stores `default` alongside `optional`; when `optional` is omitted it derives `optional := default !== undefined`. `buildFullArgs` wraps the validator with `v.optional` when optional so the Convex action accepts `undefined`.
- `validateArgs` (CLI-side argv parsing) substitutes the default for empty input too, so both the CLI and the action see the defaulted value.

## Chat agent — tool inventory must match the prompt

### MUST
- Edit BOTH `tools/generated/registry.ts` and the per-app prompt (`apps/user/server/prompt.ts`) in the same commit when wiring a new tool or provider. Why: the agent reads the system-prompt briefing, not the skill blob directory.
- Name in each prompt tool section the command, when to call it, and the answer-rendering shape. Why: an undescribed command is a command the agent never reaches.
- Keep the PERSONAL-vs-KNOWLEDGE question split in the briefing. Why: corpus-only is a knowledge-question rule, never a refusal-everything rule.
- Name every file under `apps/backend/convex/**` in camelCase (`trainingUrgency.ts`) or snake_case. Why: Convex module-path syntax bans hyphens in filenames.

### NEVER
- Let `tools/generated/registry.ts`, `tools/_app/skill.ts`, and the per-app prompt drift. Cost: three-way drift silently kills a question family — agent replies "Not in the corpus" instead of running `training status`.
- Place a hyphenated filename under `apps/backend/convex/**`. Cost: deploy fails `InvalidConfig: lib/training-urgency.js is not a valid path to a Convex module. Path component training-urgency.js can only contain alphanumeric characters, underscores, or periods.`
