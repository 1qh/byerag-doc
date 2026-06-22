# two-apps-admin-and-user

`apps/admin/` and `apps/user/` are sibling Next.js apps in the monorepo. One Convex backend serves both via the `chats.app` discriminator. Role is the app the user signed into.

## Beats

- **Single app with role-toggle UI**: admin endpoints sit at the same origin as user endpoints; XSS on user surface escalates to admin. Two origins = harder to bridge.
- **Two completely separate repos**: convergence cost on shared chat plumbing dominates; one monorepo with a shared `packages/react` is much smaller.
- **Two completely separate backends (admin Convex + user Convex)**: doubles the ops surface; cross-app queries (admin sees user activity) become RPC instead of DB joins.

## Real cost

- Two Next.js dev servers in local development.
- Two OAuth callback URIs registered in the Google Cloud Console.
- Two `apps/<name>/server/index.ts` config files (each <20 lines; trivial).
- Cookie cross-pollination prevented by origin separation; user has to sign in twice if they're an admin who also wants user-side chat.

## Gotcha for Claude

- `chats.app` is a strict union `'admin' | 'user'`. Cross-app chat is a violation of the role model — never permit.
- Both apps consume the same Convex deployment and the same `packages/react` shared chat lib. Diffs are mostly route-level + role-gated panel visibility.
- The substrate reference's `apps/_apps.ts` manifest is the place to register apps; both `admin` and `user` configs export the same `AppConfig` shape with different `id`, `buildSystemPrompt`, and `cliProviders`.
- Audit log query is gated to role `admin` inside the Convex action; admin app's audit page calls it directly, user app has no UI for it (and the action 401s if called from a user-role session anyway — defense in depth).

## MUST
- Place standalone pages (dashboard, `/test-questions`, `/docs` viewer) OUTSIDE the `(main)` route group with their own `layout.tsx` (Auth gate + `AdminSidebarNav` aside, no chat shell). Why: the `(main)` shared layout includes the chat shell.
- Apply the standalone-group pattern to the user app too, checking it first. Why: a user route under the chat shell shows non-tech employees a chat composer under `/training` `/docs`; user is the more non-tech audience.
- Guard `/chat/:id` not-found in `default-main-layout` via `useQuery(api.chats.status, {chatId})`, redirecting only when `chatStatus !== undefined && chatStatus.title === ''`; pair with `router.push`. Why: server-authoritative not-found, unblocks "Ask about this" + hard-nav landings.

## NEVER
- Gate `/chat/:id` routing on sidebar-list membership (`chats.some(c => c._id === activeChatId)`). Cost: `useChatList` is paginated and lags fresh chats → redirects every brand-new chat to `/` until the list catches up.

## Pitfall
- The `globalThis.location.assign` workaround is unnecessary once the `api.chats.status` guard is correct.
