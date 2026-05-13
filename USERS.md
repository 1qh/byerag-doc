# USERS

Two roles, two apps. Role tagged on user account, not on app origin.

## admin

Operates shared corpus. Uploads documents visible to every team member. Manages users (sets departments, promotes/demotes roles, sees who has signed in). Reads audit log. Curates assessment-test questions. Edits corpus policy. Decides scan-override on flagged uploads. Sees the dashboard.

## user

Operates own corpus + reads shared. Uploads own documents (visible only to self). Chats with the agent over shared + own scope. Takes assigned + self-assessment tests. Belongs to one of HR / Sales / IT (or unset).

## Role mechanism

Role lives on `userProfiles.role: 'admin' | 'user'` per `docs/adr/role-on-user-profile.md`.

- Bootstrap admin: env `BOOTSTRAP_ADMIN_EMAIL` (operator-set in `apps/backend/.env`). First sign-in by that email seeds `role='admin'`.
- All other first sign-ins default to `role='user'`.
- Existing admins promote/demote others via admin UI (impl deferred).
- Admin app routes (`/admin/*`) guarded by `userProfiles.role === 'admin'`. Mismatch → 403.
- User app routes accept any signed-in account.

Credential (Google OAuth) authenticates the human; account-stored role decides access.

## Department

Each `role='user'` account carries a department: HR / Sales / IT / null (per `docs/adr/departments.md`). Admin sets via admin UI. Department is a dashboard-filter dimension only — does NOT gate assignment scope.
