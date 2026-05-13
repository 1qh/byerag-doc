# schema-spec-fields-canonical

Canonical conventions across schema fields. Avoids ambiguity between docs.

## Identity fields

- `owner`, `userId`, `uploadedBy` — lowercase email per `owner-id-canonical-email-lowercase.md`.
- `chatId`, `docId`, `messageId` — Convex `Id<'table'>`; opaque string in transport.
- `sandboxId` — Docker container id (12-char hex prefix or full 64-char id; accept both, store full).

## Time fields

- `_creationTime` — Convex-provided; epoch ms; never written explicitly.
- `updatedAt`, `lastUsedAt`, `uploadedAt`, `streamingStartedAt`, `deletedAt`, etc. — epoch ms; written by mutations.
- `dayKey` — `YYYY-MM-DD` UTC string for `ownerSpend`.

## Content fields

- `content` (messages, streamEvents) — JSON string; type-discriminated by sibling `type` field.
- `extractedText` (docs) — raw utf-8 string.
- `summary` (docs) — utf-8 string; optional.

## Embeddings

- `embedding: v.array(v.float64())` — exactly 768 elements; assert length on insert.
- `dim` query param at search time — 256 / 512 / 768 (Matryoshka prefix); truncates the stored vector at query.

## Scope + role enums

- `chats.app: v.union(v.literal('admin'), v.literal('user'))`.
- `docs.scope: v.union(v.literal('shared'), v.literal('mine'))`.
- `docs.scanStatus: v.union(v.literal('pending'), v.literal('clean'), v.literal('quarantined'))`.

## Index naming

`by_<field>` for single-field, `by_<field1>_<field2>` for compound. Compound order matches query usage (most-selective first when possible).

## Beats

- **Inconsistent identity types (some email, some uid)**: agent + UI code carries conversion shims everywhere.
- **Mixed time formats**: comparisons need parsing.

## Gotcha for Claude

- Don't use `email` as a field name (lowercase implied) — use `owner` / `uploadedBy` for the relational role; reserve `email` for the auth-table copy.
- Never write `Date.now()` into a field typed `string`. Convex type system catches it, but the intent is clearer with `number`.
- `Id<'table'>` IDs in API responses serialize as strings; consumers must NOT parse them — they're opaque.
