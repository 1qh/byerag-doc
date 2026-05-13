# owner-id-canonical-email-lowercase

The string identifying a user across the system is the lowercase form of their Google account email. Stored in `chats.owner`, `docs.owner`, `docs.uploadedBy`, `userContexts.userId`, `auditLogs.owner`, `ownerSpend.owner`, `rateLimits.owner`, `sandboxes.owner`.

## Why email, not opaque user id

- Email is the natural human identifier for an internal team.
- Convex Auth surfaces email via `getAuthUserIdentity()` directly.
- Audit log lines naming `alice@org.example` are immediately readable.

## Why lowercase

- Google OAuth returns email in user-provided case; some users sign in with mixed case.
- Lowercase canonicalization at the auth callback prevents `Alice@org.example` vs `alice@org.example` from forking the same human into two owners.

## Implementation

- In `apps/backend/convex/authHelpers.ts`, after `getAuthUserIdentity`, normalize: `const owner = identity.email?.toLowerCase()`.
- All writes use the normalized form.
- Tests assert mixed-case sign-in → DB row contains lowercase.

## Beats

- **Convex `users._id` as owner**: opaque id; harder to read in audit logs; requires extra join for any UI listing users.
- **Original-case email**: forks identity on capitalization differences.
- **Hash of email**: privacy gain at internal team scale is negligible; ergonomic loss is real.

## Real cost

- A user who legally has mixed case in their email (rare, RFC permits) still gets normalized. Acceptable for internal team; email RFCs treat the local part as case-sensitive in theory, but Google + most major providers treat it as case-insensitive in practice.

## Gotcha for Claude

- Schema fields are typed `string`; the lowercase invariant is asserted at the auth boundary, not at the DB layer. A code path that writes raw `identity.email` without normalization is a bug.
- Comparisons in queries (`q.eq('owner', email)`) MUST pass the lowercase form. Helper `normalizeOwner(email)` in `authHelpers.ts` centralizes the rule.
- Sign-in audit log records the original-case email (for forensics) plus the normalized one (for joining); both fields on `auditLogs` rows.
