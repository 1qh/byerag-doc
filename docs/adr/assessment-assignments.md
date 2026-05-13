# assessment-assignments

Admin assigns a topic's assessment test to all role=user accounts in one click. Assignment fires immediately via Convex reactive WebSocket; per-assignment badge appears on each user's UI within seconds. Badge persists until that user passes the topic via an assigned-kind attempt. Admin can un-assign at any time; un-assignment nukes all assignment rows silently. Pool < 5 blocks creation; admin sees the gate at click time.

## Trigger and scope

Admin clicks `Assign` on a topic in `/admin/assignments` or `/admin/topics/<id>`. Server:

1. Verifies caller's role is `admin`.
2. Verifies topic exists, not deleted, approved pool size ≥ 5. If not, returns 400 with reason; no rows created.
3. Walks the user list (role=user only; admins excluded).
4. For each user, checks `testPasses.by_user_topic_kind` for an existing `(userId, topicId, kind='assigned')` row. If exists, skip — user already passed this topic via a prior assignment.
5. For each non-skipped user, inserts a `testAssignments` row with `userId`, `topicId`, `createdAt=now()`, `createdBy=admin email`, `deletedAt=null`.
6. Returns count of new assignments created + count of skipped users.

`testAssignments.by_user_topic` index ensures O(1) badge lookup per user per topic. Convex reactive subscription on `testAssignments.by_user` (filter `deletedAt=null AND no matching testPasses`) gives each user's pending badge list in real time.

## Badge UI

Per the locked rule, each assignment is a separate persistent badge — not aggregated.

- Sidebar nav shows a count `Training (N)` where N = pending assignments for this user.
- Click → assignments list, one row per pending assignment, each showing topic name.
- Per-assignment badge cleared the moment a passing assigned-kind attempt lands for that (user, topic).

UI mechanism details deferred per the rule "UI/UX discussed at coding part." Behavior contract above is canonical.

## Re-fire vs skip

A second admin click `Assign Security to all` after some time:

- Users who passed the prior assignment AND have NOT been re-armed (per `re-arm-on-substantive-corpus-update.md`) → skipped, no new row.
- Users who passed but were re-armed (their `testPasses.assigned` was deleted) → fresh assignment row, fresh badge.
- Users who never passed an assigned-kind → see their pending assignment row remain (idempotent; no duplicate insert because `(userId, topicId)` already has a non-deleted row).

Server is idempotent on `(userId, topicId)`; never inserts a duplicate assignment row.

## Un-assignment

Admin clicks `Cancel assignment` on a topic. Server:

1. Verifies caller's role is `admin`.
2. Walks `testAssignments.by_topic` where `deletedAt=null`.
3. For each row, sets `deletedAt=now()`. (Hard-delete vs soft-delete: rows soft-deleted for audit; user-visible reactive query filters `deletedAt=null` so badges disappear immediately.)
4. Cascades: any `in-progress` `testAttempts` row with `kind='assigned'` for the affected (user, topic) flips to `cancelled`.
5. Writes one `auditLogs` row with `command='training.assignment.cancel'`, `args={topicId, affectedCount}`, `severity='medium'`.

No notification fires to users about the cancellation. Badges silently vanish via reactive sub.

Past `testPasses` records remain. A user who already passed before cancellation keeps their pass-state.

## Real-time fire

Convex reactive subscription on `testAssignments.by_user_pending` (computed view) pushes the new row to the user's open browser within seconds of the admin insert. No polling, no client refresh required.

Offline users: assignment row already exists; on next sign-in, the subscription replays it.

## Admin exemption

Admins (`role='admin'`) are excluded from `Assign to all users` automatically. Admin curates the content; testing them on questions they approved is theater. If an admin wants to self-test, they sign in to the user app with a user-role account OR self-assess from admin app's `/training` page (admin app's training surface, if any, deferred to UI/UX discussion).

## Pool gating

Pool < 5 blocks assignment creation entirely. Server returns 400. Admin sees `Topic 'X' has 3 approved questions; need at least 5 before assigning.`

Existing assignments for a topic that later loses questions below the threshold (e.g., admin retires several) remain alive — pre-existing badges don't vanish. But new attempts can't be started (per `assessment-test-attempts.md` pool gate); user sees `Start` disabled until admin restores the pool.

## Schema

See `SCHEMAS.md` for `testAssignments` and `testPasses` table definitions and indexes.

## Audit

- Bulk assign: `auditLogs` row with `command='training.assignment.create'`, `args={topicId, count}`, `severity='medium'`.
- Bulk cancel: `auditLogs` row with `command='training.assignment.cancel'`, `args={topicId, count}`, `severity='medium'`.

## Gotcha for Claude

- Real-time fire is via Convex reactive sub on a JS-defined computed view; not a DB trigger. The view selects `(testAssignments.by_user)` joined against `(testPasses.by_user_topic)` to surface pending assignments only.
- Reactive sub costs O(active subscribers × delta rows) Convex bandwidth. At internal-team scale, trivial; at 10k+ users, revisit query shape.
- Admin un-assign cascade to in-progress attempts is critical; otherwise user sees `Start` disabled (because assignment-canceled), but their open attempt UI keeps polling and gets confused. Cascade flips attempt status atomically with the assignment soft-delete.
- Idempotency of re-fire: server uses a single mutation per (user, topic) tuple; even rapid double-clicks by admin produce at most one row.
- Skip logic checks `kind='assigned'` only on `testPasses`. A user who passed only via self-assessment is NOT skipped; they get a fresh assignment per the locked self-pass ≠ assignment-pass rule.
