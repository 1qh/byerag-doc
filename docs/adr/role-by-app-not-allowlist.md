# role-by-app-not-allowlist

Role is determined by which app the user signed into. Admin app → role `admin`. User app → role `user`. No `ALLOWED_EMAILS` allowlist in env.

## Beats

- **Email allowlist (`ALLOWED_EMAILS` env)**: drift between env and reality; new team members require an env edit + Convex sync; revocation by removing from env is reactive, not auditable.
- **Single app with role-toggle in UI**: blast radius bigger — admin endpoints sit on the same origin as user endpoints; XSS on user surface escalates.
- **Per-user role table seeded manually**: still requires manual seeding; same drift problem.

## Real cost

- Two apps to maintain (mitigated by `packages/react` shared chat plumbing — the apps are thin shells).
- Two Next.js dev servers in local development.
- Two OAuth callback URIs in the Cloud Console.

## Gotcha for Claude

- Network ACL is the access gate. Admin app must be reachable only from admin-trusted network segments (host firewall rule restricting the admin-app port, or VPN-only). Without that, "role-by-app" collapses to "anyone who guesses the admin URL".
- Convex's auth callback handler maps origin → role at session-write time. Session is then trusted DB-side. Role is NOT re-derived on every query — it's stamped once.
- Sign-in via admin app does NOT grant admin role on user app (separate cookie origin). Same human signs in twice if they need both.
- Audit log records `(user, app, role, action)` on every privileged operation. Admin can query; user cannot.
