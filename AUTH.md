# AUTH

Two apps, one OAuth client, role-by-app.

## Provider

Google OAuth via `@convex-dev/auth`. One OAuth client registered in Google Cloud Console with both apps' callback URIs whitelisted. Client id and secret live in operator-local `.env`, synced to Convex env via `sync.ts` per pm4ai convention.

## Callback URIs registered on the client

- `http://localhost:<admin_port>/api/auth/callback/google`
- `http://localhost:<user_port>/api/auth/callback/google`
- `http://localhost:3210/.well-known/openid-configuration/callback/google` (Convex site URL)

Concrete port numbers live in operator-local env, not in this spec.

## Role determination

Role is the app the OAuth callback originated from:

- Sign-in completed on `apps/admin` → role `admin` stamped on the session.
- Sign-in completed on `apps/user` → role `user` stamped on the session.

The Convex auth callback handler reads the origin/redirect URL, maps to `app: 'admin' | 'user'`, writes the role into the session metadata. Subsequent Convex queries read role from the session.

No `ALLOWED_EMAILS` allowlist. Network ACL is the gate — only people who can reach the admin app's URL can sign in there.

## Session lifecycle

Standard `@convex-dev/auth` cookie session. Sign-out clears the cookie. No refresh-token rotation required at internal-team scale.

## JWT keypair

`@convex-dev/auth` requires `JWT_PRIVATE_KEY` + `JWKS` env vars. Generated once on first compose boot of the local Convex backend, persisted to operator-local `.env`. Never regenerated unless the deployment is wiped.

## CLI auth

Separate from web auth. CLI tokens live in `cliTokens` table:

- **Device flow** (`bunx byerag login`): opens browser tab, user authorizes in the admin or user app, token returned.
- **PAT** (admin-issued): admin generates a labeled token for a service / CI, copies once, never retrievable again.

CLI tokens are scoped to the user; calls hit `/api/cli/exec` with `Authorization: Bearer <token>`. The proxy resolves the token → user → role → permitted commands.

## Audit

Every sign-in writes to `auditLogs` via the Convex auth callback. Every CLI exec writes to `auditLogs`. Admin can query the log via a Convex query exposed only to role `admin`.
