# departments

User accounts (`role='user'`) carry one of three department tags: `HR | Sales | IT`. Admins carry no department. Single department per user. Dashboard filter affordance only — assignment scope still always-all-users in v0.

## Schema

New table `userProfiles`:

- `userId: string` — lowercase email
- `department: 'HR' | 'Sales' | 'IT' | null` — null when role=admin
- `updatedAt: number`
- `updatedBy: string` — admin email who set/changed

Index: `by_userId`.

Departments are NOT a permission gate. Admin still sees + does everything regardless of dept. Users see their own dept on profile only.

## Admin sets department

When a new user first signs in, `userProfiles.department = null`. Admin sets via `/admin/users` page (defer impl UI). Until set, user is excluded from gradebook department-filter slices (shown under "Unassigned" group).

## Beats

- **Per-department admin role** — over-engineered; one global admin sees all.
- **Department-scoped assignments** — deferred; locked rule = always-all-users.
- **Email-domain auto-assign** — fragile; team uses Google Workspace with mixed domains.
- **Free-form text department** — drift; closed enum stable.

## Real cost

- One field + one table.
- Admin manually sets department per user once.
- Topics don't carry dept affinity (defer).

## Gotcha for Claude

- Department doesn't gate assignments. `Assign to all` includes all role=user regardless of dept. Agent cron does same.
- Admin's role check is the access gate; department is metadata only.
- The Training page Users table shows department as a column (null → "Unassigned"); pass-rate math unchanged.
- Future v1: when department-scoped assignments land, this table is ready — just extend `testAssignments` flow to filter on `userProfiles.department`.
