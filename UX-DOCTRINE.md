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

Click a doc → preview pane (markdown render for `.md`, PDF.js viewer for PDFs, plain-text fallback for others). Keyboard nav: arrow keys to switch docs, slash to focus search.

## Upload widget

Drag-and-drop or file picker. Multi-file accepted. Shows per-file scan status (pending / clean / quarantined). On clean: appears in the doc list immediately via Convex reactive subscription. On quarantined: red banner with reason.

## Admin-only surface

Admin app has one extra section: `Audit`. A paginated table over `auditLogs` filterable by owner / command / time. No edit actions — read-only by design.

## Look

shadcn components as-is per `book/HARD-RULES.md`. Semantic colors only (`text-foreground`, `bg-primary`, `text-destructive`). `cn()` for conditional classes. Minimal DOM.

## Two-app convergence

Differences between admin and user apps are routes + role-gated panels, not separate UI primitives. Both apps consume the same `packages/react` components.
