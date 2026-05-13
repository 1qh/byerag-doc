# dashboard-admin-landing

Admin app's default landing route is `/admin/dashboard`. Sign-in → dashboard. Doc library reached via nav click.

## Beats

- **Doc library as landing** — admin's morning routine is overview check, not upload.
- **Last-visited route** — cognitive load; admin can't predict where they land.

## Real cost

- Route precedence.
- Dashboard must load fast (live tiles via reactive sub).

## Gotcha for Claude

- Sign-in callback redirects to `/admin/dashboard`.
- Deep-link to specific tile via URL fragment (e.g. `/admin/dashboard#cost`); future scope.
- Dashboard is admin-only; user app has no equivalent (users see `/training` as main surface).
