# docs-table-via-convex-storage

Document blobs live in Convex `_storage`. Metadata in `docs` table includes `storageId: Id<'_storage'>`. ACL columns (`scope`, `owner`) gate visibility. Embedding column carries the nomic-embed vector when computed.

## Beats

- **MinIO / S3-compatible external blob store**: another service to run, back up; ACL becomes cross-system. Convex `_storage` keeps blobs in the same transactional boundary as the metadata row.
- **TigerFS (Postgres-rows-as-files via FUSE)**: cute but immature; per-file size limits (~750MB binary via base64 in TEXT) constrain real-world doc sizes; FUSE adds a layer with its own ops surface. Convex `_storage` is the boring path.
- **Direct filesystem on the host**: ACL becomes filesystem perms; cross-replica filesystem sync becomes an ops problem. Convex blob is the simple shape.

## Real cost

- Convex `_storage` blob size cap (currently generous but documented as bounded; check the self-host version).
- Reading a blob inside a Convex action loads the full bytes into memory. Streaming large docs to the sandbox is a separate code path (sandbox pulls via a signed URL or chunked read).

## Gotcha for Claude

- `_storage` ids are first-class Convex ids: `Id<'_storage'>`. Don't pass them as plain strings; type system catches the mismatch.
- Soft-delete the doc row first (`deletedAt`), then schedule storage.delete in a background action after a grace period — accidental deletes are recoverable for a window.
- Extracted-text-cache field on `docs` (P3) avoids re-extracting on every read; populated by a background action after upload + scan.
- Vector index filter fields are fixed at index creation (`['owner', 'scope']`); adding a new filter requires a schema migration + index rebuild.
- ACL: every Convex query/action returning a `docs` row checks `row.scope === 'shared' || row.owner === caller.email` before return. Centralize in a `assertDocVisible(row, caller)` helper.
