# doc-versioning-and-deletion-cascade

Same-filename re-upload creates a new `docs` row with `version: N+1` and a `supersedes: Id<'docs'>?` link to the prior row. The prior row's `supersededBy` is set in the same transaction. Soft-delete via `deletedAt`; hard-purge of `_storage` blob runs 30 days later via scheduled function.

## Schema additions to `docs`

- `version: number` — 1-based; default 1 on first upload
- `supersedes: Id<'docs'>?` — previous version
- `supersededBy: Id<'docs'>?` — next version (mirror, set in the same txn that creates the next)
- `deletedAt: number?` — soft-delete tombstone

## Versioning rule

Same (`scope`, `owner`, `filename`) → bump version. Different scope or owner → independent doc (separate version chain).

## Read behavior

`docs list --scope <s>` returns only the latest non-deleted version per filename chain (where `supersededBy == null` AND `deletedAt == null`).

`docs read --id <id>` returns any version (latest or historical). The `id` is the surface; if a chat citation points at an old version, the answer renders with a "this doc was updated on YYYY-MM-DD" badge.

## Deletion cascade

- Soft-delete sets `deletedAt = now()`. Doc invisible in `docs list`; chat citations still render (with "deleted" badge).
- Embeddings in `docChunks` not eagerly removed; they remain searchable until hard-purge.
- Hard-purge scheduled function runs 30 days after `deletedAt`: deletes `docChunks` rows, deletes `_storage` blob, NULLs the row's `storageId` (keeps the row for audit chain).

## Beats

- **No versioning, last-write-wins**: chat citation to "Q3 contract" rots silently when an updated version is uploaded.
- **Eager hard-delete**: irreversible mistakes lose data.
- **Per-version `_storage` blob retained forever**: storage cost grows; 30-day purge is the tradeoff.

## Real cost

- One more row per re-upload (no blob duplication unless content actually differs — sha256 dedup on `_storage` write).
- Scheduled function fires daily, walks `docs.by_deletedAt`, hard-purges expired entries.

## Gotcha for Claude

- Citation rot: a chat answer cited `docId X`; the doc gets a new version. The citation still resolves (the prior version stays available); UI shows a chip with version + supersession info. Agent can choose to re-cite the new version on follow-up.
- Schema migration: adding `version`, `supersedes`, `supersededBy`, `deletedAt` is a P0 commit. Pre-launch wipe + regenerate per `book/HARD-RULES.md`.
- `supersedes` chain length is bounded only by re-upload frequency; for forensics, walk the chain via `by_supersedes` index.
- Vector search: filter on `deletedAt == null` to exclude tombstoned docs from `docs similar` results.
