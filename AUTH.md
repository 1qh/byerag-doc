# AUTH

Google OAuth. One OAuth client. Role on user account (per `docs/adr/role-on-user-profile.md`). Admin app gates on role at the route layer.

## Provider

Google OAuth via `@convex-dev/auth`. One OAuth client registered in Google Cloud Console. Callback URIs cover both apps + Convex site. Client id and secret live in operator-local `.env`, pushed to Convex env via `apps/backend/sync.ts`.

## Callback URIs

- `http://localhost:<admin_port>/api/auth/callback/google`
- `http://localhost:<user_port>/api/auth/callback/google`
- `http://localhost:3210/.well-known/openid-configuration/callback/google` (Convex site URL)

Concrete port numbers operator-local.

## Role determination

Role is `userProfiles.role: 'admin' | 'user'`.

- First sign-in: `userProfiles` row created. Default `role='user'`.
- Bootstrap admin: env `BOOTSTRAP_ADMIN_EMAIL` (operator-set, comma-separated for multi). First sign-in matching gets `role='admin'` seeded.
- Existing admins promote/demote others via admin UI (impl deferred).
- Admin app `/admin/*` routes guarded; non-admin → 403.
- User app: any signed-in account.

No `ALLOWED_EMAILS` env, no VPN gating.

## Session lifecycle

Standard `@convex-dev/auth` cookie session. Sign-out clears the cookie. Role changes (promote/demote) take effect on next session refresh (operator must sign-out/sign-in OR a session-rotate hook fires).

## JWT keypair

`@convex-dev/auth` requires `JWT_PRIVATE_KEY` + `JWKS` env vars. Generated once on first compose boot of the local Convex backend; persisted to operator-local `.env`. Never regenerated unless the deployment is wiped.

## CLI auth

Separate from web auth. CLI tokens live in `cliTokens` table:

- **Device flow** (`bunx byerag login`): opens browser tab, user authorizes via Google OAuth, token returned.
- **PAT** (admin-issued): admin generates a labeled token for service / CI; plaintext shown once.

CLI tokens are scoped to the user; calls hit `/api/cli/exec` with `Authorization: Bearer <token>`. Server resolves token → user → `userProfiles.role` → permitted commands.

## Audit

Every sign-in writes to `auditLogs` via the Convex auth callback. Every CLI exec writes to `auditLogs`. Admin queries the log via a Convex query gated by `role='admin'`.
