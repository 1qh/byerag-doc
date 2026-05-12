# CLI-SURFACE

Canonical agent-facing CLI commands. Each command is a Convex action registered in the tool registry; the CLI binary is a thin dispatcher.

Pattern: `byerag <category> <verb> [--flag value ...]`.

## docs list

List documents in scope. Returns id, filename, mime, size, uploadedAt, scope.

- `--scope` (union) required enum: `shared | mine | both`
- `--limit` (number) — default 50, max 200

## docs read

Fetch a document's extracted text (or first N bytes for binary fallback).

- `--id` (string) required — `docs._id`
- `--bytes` (number) — cap on returned bytes (default 200_000, max 2_000_000)

ACL: row is fetched, if `scope='mine'` and `owner != caller.email` → `FORBIDDEN`.

## docs grep

Regex match across docs in scope. Returns matching `(docId, lineNumber, snippet)` tuples.

- `--pattern` (string) required — regex (re2 dialect)
- `--scope` (union) required enum: `shared | mine | both`
- `--limit` (number) — default 50

Implementation: iterate `docs` rows in scope (paginated), pull extracted text from `_storage` (or cached text field), run regex, return hits.

## docs diff

Unified diff between two docs (or two versions of one doc when versioning lands).

- `--a` (string) required — `docs._id`
- `--b` (string) required — `docs._id`
- `--context` (number) — lines of context, default 3

ACL: both rows checked individually.

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

## Output shape

All commands return JSON on success. On error: `{error: {category, code, message, retryable, details?}}` and non-zero exit. Categories: `auth | input | permanent | transient | upstream`.
