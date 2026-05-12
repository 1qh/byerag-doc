# google-oauth-via-convex-auth

Google OAuth provider wired via `@convex-dev/auth/server`. One OAuth client registered in Google Cloud Console, callback URIs whitelisted for both apps' origins + Convex site URL.

## Beats

- **Authentik / Keycloak self-host**: another service to run, manage, back up. Internal team scale doesn't need the surface area.
- **Basic password (own users table)**: friction for the user, no SSO story.
- **Magic link via internal SMTP**: SMTP setup at small scale is its own rabbit hole.

## Real cost

- Vendor dependency on Google's OAuth endpoint for sign-in (acceptable: `accounts.google.com` is not a data-residency concern; only the access token traverses, not doc content).
- One-time Cloud Console UI step to register callback URIs.
- Client id + secret live in operator-local `.env`.

## Gotcha for Claude

- Add every dev port to the OAuth client's authorized redirect URIs before testing locally — `OAuth Error: redirect_uri_mismatch` is the loudest failure mode.
- `SITE_URL` in Convex env must include every origin that can issue OAuth callbacks (admin app, user app, Convex site URL).
- Convex auth callback writes to `authTables` (users, accounts, verification tokens); session cookie set on the calling origin. Cross-origin cookies do NOT leak between admin app and user app — that's the role-isolation invariant.
- `@convex-dev/auth` doesn't natively expose "which app callback this sign-in came from". Custom callback handler reads the `redirectTo` URL, maps to `app: 'admin' | 'user'`, writes role into the session metadata. See `AUTH.md` for the mapping shape.
