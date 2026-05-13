# upload-dedup-and-version-prompt

Upload flow surfaces three end-user outcomes: rejected (malicious), duplicate (same content already exists), version-conflict (same filename, different content). The first two are automatic; the third is an interactive prompt.

## Outcome matrix

After scan-clean + sha256 computed, the upload action checks `docs.by_sha256_scope_owner`:

| Lookup result | Outcome | UI |
|---|---|---|
| sha256 + scope + owner match existing non-deleted row | **duplicate** | toast: "this file is already in your library (uploaded as `<filename>` on `<date>`)." Reject; no new row. |
| filename + scope + owner match an existing non-deleted row, sha256 differs | **version-conflict** | modal: "a different file with this name already exists. Replace it?" Buttons: Replace · Keep both · Cancel |
| neither matches | **clean insert** | new `docs` row, version=1 |

## "Replace" semantics

Replace → server-side mutation in one transaction:
- Insert new row with `version = previous.version + 1`, `supersedes = previous._id`.
- Set previous row's `supersededBy = new._id` and `deletedAt = now()`.
- Scheduled hard-purge of the previous version's blob runs 30 days later per `doc-versioning-and-deletion-cascade.md`.

Citations to the previous version still render (with badge) until hard-purge.

## "Keep both" semantics

Server adds a numeric suffix to the new filename (`contract.pdf` → `contract (2).pdf`); new row created with `version=1` and no supersession link. Both rows are independent docs.

## "Cancel" semantics

No row inserted; the staged blob in the upload service's tmp dir is deleted. Same effect as if the user closed the modal.

## Schema additions

- `docs.by_sha256_scope_owner` compound index: `(sha256, scope, owner)`.
- `docs.by_filename_scope_owner` compound index: `(filename, scope, owner)`.

Both used for the duplicate / version-conflict check in one query each.

## Cross-scope behavior

A user-uploaded doc that's already in shared scope: still a clean insert in `mine` scope (different scope → different row). Rationale: the same file may legitimately exist as both an org-wide reference AND a user's personal annotated copy.

A doc admin uploads to shared scope that already exists in some user's `mine` scope: still a clean insert in `shared` scope. Same rationale, mirror direction.

## Quarantined-on-scan-fail

Per `book/SECURITY.md`: scan-fail produces a `docs` row with `scanStatus='quarantined'` + null `storageId` + no `extractedText`, no `embedding`. UI surfaces:

> ⚠️ Your file was rejected because it appeared suspicious. Reason: `<scanner signature>`.

Audit log records the rejection with uploader + sha256 + signature. Repeated rejected uploads of the same sha256 from the same owner within 1 hour rate-limit additional attempts (`429 too many rejected uploads`).

## Beats

- **Auto-version-bump on filename collision (no prompt)**: surprises admin; "I just renamed and now my old version is gone" is the loudest failure mode.
- **Block any same-filename upload**: workflow blocker; admins legitimately need to update docs.
- **Content-dedup at the storage layer only (Convex `_storage` sha-keyed)**: storage dedups, but the metadata row still gets created, polluting `docs list`.

## Real cost

- Two index lookups per upload (sha256 hit OR filename hit; rare to hit both at once).
- One modal in the UI for the version-conflict case.

## Gotcha for Claude

- `sha256` is over the raw bytes post-scan-pass; identical content uploaded by two different admins still dedups (within the same `(scope, owner=null)` partition for shared).
- The version-conflict prompt is BLOCKING per file in a multi-file upload; show a queue UI so admin can resolve each conflict before the next file is staged.
- "Replace" is reversible within 30 days (the previous blob's hard-purge schedule). Surface a small "undo" toast right after Replace lands.
- For the user app: same prompt UX. User can also "Replace" or "Keep both" on their own `mine` scope.
- Content dedup runs against non-deleted rows only. A deleted-but-not-yet-purged row does not block re-upload of the same content (re-upload of a deleted file means the user changed their mind; insert a fresh row).
