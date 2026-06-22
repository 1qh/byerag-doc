# nextjs-16-app-router

Next.js 16 with App Router + React 19 + Turbopack. Server components default; `'use client'` only when hooks needed.

## Beats

- **Vite + custom SSR**: rebuild streaming, middleware, auth callback plumbing from scratch.
- **Remix**: smaller community than Next; route loader shape diverges from RSC.
- **SvelteKit**: stack lock-in conflict with React-only `packages/react`.

## Real cost

- Turbopack on dev is fast; production build still goes through webpack for some shapes.
- App Router has rough edges around streaming + auth interception; resolves with `<Suspense>` discipline.

## Gotcha for Claude

- `<Suspense>` around `useSearchParams()` — required, else hydration mismatch.
- `Date.now()` / `Math.random()` forbidden in render (hydration drift).
- No raw `<button>` / `<table>` when shadcn has a component.
- `'use client'` only when needed — layout / loading / error files are server components.
- `NEXT_PUBLIC_*` env vars ship to the browser bundle. Never for secrets.

## MUST
- Name any app-config file returning JSX (e.g. `sidebarSlotAboveHistory: <AdminSidebarNav/>`) `.tsx`, never `.ts`. Why: Next 16 + Turbopack won't parse JSX in `.ts`.
- Run `sh scripts/restart-dev.sh` after any mid-session `bun i` or any `/tmp/next-panic-*.log` "Next.js package not found" panic. Why: canonical Turbopack-panic recovery.
- Restart `next dev` after editing any `NEXT_PUBLIC_*` value. Why: apps run `bun --env-file=../backend/.env next dev`; env loads at process start only.
- Set BOTH `ALLOW_DEV_TOKENS=1` (server, auth.ts provider list) AND `NEXT_PUBLIC_ALLOW_DEV_TOKENS=1` (client) for env-gated dev sign-in. Why: Anonymous provider needs server enable + client render — only Playwright-authable session path.
- Place CSP in `next.config.ts` `async headers()`, not a `proxy.ts` middleware; source matcher excludes `_next/static|_next/image|favicon.ico`. Why: a `proxy.ts` makes Next 16 emit `_clientMiddlewareManifest.js` that warns on MIME.
- Test the resolved CSP value in `default-config.test.ts`. Why: CSP is mechanism-asserted, not eyeballed.

## NEVER
- Treat a cache nuke alone as Turbopack-panic recovery. Cost: the running `next dev` holds module-graph refs to pre-install `next`; the process must restart.
- Keep a `proxy.ts` solely for CSP. Cost: Next 16 emits `_clientMiddlewareManifest.js` served `application/json` but loaded via `<script src>` with nosniff → MIME warning.

## Pitfall
- Turbopack panic surfaces as every route 500 with body `{"err":{"message":"Panic in async function"}}`; browser sees `__NEXT_DATA__` `pageProps.statusCode:500` — easy to misread as page-code bug. `/tmp/next-panic-*.log` stack: `subscribe_issues_and_diags_operation → pick_endpoint → entrypoints_without_collectibles_operation → AppProject::routes_with_filter → directory_tree_to_entrypoints_internal → directory_tree_to_loader_tree → Next.js package not found`.
- Manual recovery if `restart-dev.sh` unavailable: kill both `next dev --turbo` wrappers + `rm -rf apps/<app>/.next apps/<app>/.turbo` + restart `bun --filter=<app> dev`; cache nuke without process restart is insufficient. `restart-dev.sh` nukes `apps/admin/.next .turbo` + `apps/user/.next .turbo`, respawns detached, logs `/tmp/{admin,user}-dev.log`.
