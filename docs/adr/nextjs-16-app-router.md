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
