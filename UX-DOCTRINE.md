# UX-DOCTRINE

Both apps share UI primitives from `packages/react`. Each app composes its own shell.

## Common shell

Three panes when there's a chat selected:

```
┌──────────────┬─────────────────────────┬──────────────┐
│ Sidebar      │ Chat                    │ Docs panel   │
│              │                         │              │
│ + New chat   │ user: …                 │ [shared]     │
│              │ assistant: …            │   • doc1.pdf │
│ Threads      │   🔧 docs grep …        │   • doc2.md  │
│ • Q1         │   citations: doc1#L42   │ [mine]       │
│ • Q2         │                         │   • notes.md │
│              │ [input]                 │   [+ upload] │
└──────────────┴─────────────────────────┴──────────────┘
```

Sidebar lists chats for the current user, most-recent first. Doc panel lists docs in scope; admin app shows only `shared` upload widget; user app shows both `shared` (read-only) and `mine` (editable).

## Empty state

First-time user lands on the welcome screen with one CTA: "Upload your first doc" or "Start a chat" (after at least one shared doc exists). No marketing copy, no onboarding tour.

## Streaming render

Convex reactive subscription on `streamEvents` for the active chatId. Client buffers partial deltas (`text_delta`, `thinking_delta`, `input_json_delta`) and renders incrementally.

Tool calls render as collapsible breadcrumbs with the tool name, args summary, and result preview. Click to expand.

Citations render as inline chips `[doc1.pdf §3]`. Click → opens the doc panel scrolled to the referenced section.

## Doc panel

Citation chips in chat open the source doc in a side sheet (`<Sheet side="right">`) over the chat shell — plain left-click opens; ESC closes; chat stays interactive behind the sheet so the user can type follow-ups while reading. Modifier-clicks (cmd / ctrl / shift / middle) fall through to native `<a href>` and open `/docs/<docId>` in a new tab. The `/docs/<docId>` route remains the shareable deep link. Per `docs/adr/citation-side-sheet.md`.

## Upload widget

Drag-and-drop or file picker. Multi-file accepted. Shows per-file scan status (pending / clean / quarantined). On clean: appears in the doc list immediately via Convex reactive subscription. On quarantined: red banner with reason.

## Chat composer upload

The chat composer carries a visible attach control (paperclip in the input toolbar, beside Send) plus drag-and-drop onto the composer. Multi-file accepted. Uploaded files run the identical pipeline as the Docs-page widget (`generateUploadUrl` → blob POST → `docs.upload` finalize → scan → policy → embed → sandbox materialize); scope is `mine` in the user app, `shared` in the admin app. On success the composer appends a plain-language line naming the files and pointing the agent at the doc tools in the relevant scope (no filesystem-path framing — the files are `docs` rows, reachable via `docs list` / `docs read`, not literal `/workspace` paths). This makes "upload N files and ask the agent to compare them" a single in-chat flow; the Docs page remains the durable library view.

## Admin-only surface

Admin app has one extra section: `Audit`. A paginated table over `auditLogs` filterable by owner / command / time. No edit actions — read-only by design.

## Look

shadcn components as-is per `book/HARD-RULES.md`. Semantic colors only (`text-foreground`, `bg-primary`, `text-destructive`). `cn()` for conditional classes. Minimal DOM.

## Sidebar is the navigation SSOT

The sidebar is layout-level and single-sourced; pages only render content into `<main>`. One nav component per app (`UserSidebarNav` = Chat · Training · Docs; `AdminSidebarNav` = Dashboard · Docs · Policy · …) is the single source of truth for navigation — it is consumed by BOTH the chat shell (injected via `app.config` `sidebarSlotAboveHistory`) and the `(standalone)` layout. Add / rename / reorder a link once → every page reflects it; no per-page nav, no chat-shell vs standalone divergence.

Account + theme are universal controls, never buried in one shell: the shared `SidebarUserNav` (`@a/react/components` — avatar/email, Light/Dark, Log out) is mounted at the bottom of every shell (chat AND standalone, both apps). A user on `/training` or `/docs` can sign out or switch theme without navigating back to chat. Burying login/logout or theme behind a single route is a non-tech-UX failure.

## Two-app convergence

Differences between admin and user apps are routes + role-gated panels, not separate UI primitives. Both apps consume the same `packages/react` components.
