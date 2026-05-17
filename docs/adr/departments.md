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

- The agent auto-assign never filters by department — it keeps every eligible (user, topic) cell filled regardless of dept.
- The manual Assign composer can target a single department (HR/Sales/IT/Unassigned) — `userProfiles.department` filter, `null` → "Unassigned". Admin's role check is still the access gate; department is targeting metadata.
- The Training page Users table shows department as a column (null → "Unassigned"); pass-rate math unchanged.
