# role-on-user-profile

User account carries its role explicitly via `userProfiles.role: 'admin' | 'user'`. Admin app route guard reads the field; access requires `role='admin'`. No network/VPN segmentation as access gate. No email-allowlist env var.

## Mechanism

- `userProfiles.role` is the source of truth. One row per user account.
- First sign-in: `role` defaults to `'user'`.
- Bootstrap admin: env var `BOOTSTRAP_ADMIN_EMAIL` (operator-set in `.env`). First sign-in by that email gets `role='admin'` seeded automatically at the auth callback. Subsequent sign-ins by that email continue to read the stored role (which may have been changed by another admin).
- Existing admins promote/demote other users via admin UI (impl deferred).
- Admin app route guard: every `/admin/*` route checks `userProfiles.role === 'admin'` on the session's user. Mismatch → 403.
- User app accepts any signed-in account (including admins).

## Beats

- **Role-by-app (which app you signed into)**: anyone who reaches the admin app URL becomes admin. Network gate would be required to make this safe — adds operational dependency.
- **`ALLOWED_EMAILS` env list**: drift between env and reality; admin must redeploy/restart on every team change.
- **Network segmentation alone**: defense in depth requires multiple layers; account-level role + network constraints can coexist but role-on-account is the load-bearing layer.

## Real cost

- One field on `userProfiles`. One bootstrap env var.
- Promote/demote UI is deferred but the data model supports it from day 1.

## Bootstrap flow

1. Operator sets `BOOTSTRAP_ADMIN_EMAIL=admin@x.com` in `apps/backend/.env`.
2. `bun sync` pushes to Convex env.
3. Operator (admin@x.com) signs in for the first time on either app.
4. Auth callback creates `userProfiles` row with `role='admin'` because email matches `BOOTSTRAP_ADMIN_EMAIL`.
5. Admin can now reach `/admin/*` routes.

If bootstrap env unset OR no one matching has signed in yet, no admin exists; admin routes return 403 for everyone. System still functional for user-app sign-ins.

## Gotcha for Claude

- The `BOOTSTRAP_ADMIN_EMAIL` env is consulted ONLY at auth-callback `userProfiles` row creation. Changing the env after the bootstrap admin has signed in doesn't promote a different email automatically — admin must use the UI to demote/promote.
- Multiple bootstrap admins: `BOOTSTRAP_ADMIN_EMAIL` accepts a comma-separated list. Each email gets admin on first sign-in.
- Demoting yourself: admin UI must prevent self-demotion if it would leave zero admins. Otherwise system locks out.
- Admin's session role is cached on the session token; promote/demote takes effect on next session refresh (sign-out + sign-in OR auto-rotate hook). Document this UX caveat to admin.
- Role and department are orthogonal. Admin has `role='admin'` AND `department=null`. User has `role='user'` AND `department ∈ {'HR','Sales','IT', null}`.
