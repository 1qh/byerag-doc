# citation-side-sheet

Citation chips in chat open the source doc in a shadcn `<Sheet side="right">` over the chat shell. Plain left-click opens the sheet; modifier-clicks and middle-click fall through to native `<a href>` (new tab). The `/docs/<docId>` route stays as the shareable deep link.

Lives in `packages/react`, so admin and user apps inherit identical behavior.

## Beats

- **Same-tab navigation to `/docs/<docId>`**: unmounts the chat shell, loses scroll, requires browser back. Breaks the "verify the citation while continuing the conversation" flow.
- **Modal dialog overlay**: centers focus but blocks the chat input underneath. Worse for follow-ups.
- **Always-on three-pane (chat left, doc right)**: wastes screen real estate when the user isn't actively verifying. Sheet is the on-demand version of the same idea.

## Mechanism

- `DocSheetProvider` mounted in `packages/react/src/next/default-providers.tsx`. Exposes `openDoc(docId, anchor?)` and `close()` via `useDocSheet()`.
- `<DocSheet>` rendered once inside `default-main-layout.tsx`. Reads provider state, renders shadcn `<Sheet side='right' modal={false}>` with shared `<DocViewer>` inside.
- `<DocViewer>` lives in `packages/react/src/components/doc-viewer.tsx`. Both the route page (`/docs/<docId>`) and the sheet import the same component.
- `<CitationAnchor>` intercepts left-click only when no modifier keys are held (`e.button === 0 && !metaKey && !ctrlKey && !shiftKey && !altKey`); calls `openDoc`. All other click variants pass through to native `<a href>`.
- Sheet is non-modal: chat textarea stays focusable, user can type follow-ups while the doc is open beside them.
- `useQuery(api.docs.read, ...)` only fires when `selectedDocId` is non-null — closed sheet does no fetching.

## Real cost

- One shadcn Sheet primitive (already in `readonly/ui/`).
- `<DocViewer>` moves from admin route page to shared package (~30 lines).
- Click interceptor on `<CitationAnchor>` (~5 lines).
- Both apps consume `DefaultMainLayout` from `packages/react` — zero per-app delta.

## Gotcha for Claude

- Intercept ONLY plain left-click. Check `e.button === 0 && !e.metaKey && !e.ctrlKey && !e.shiftKey && !e.altKey` before calling `e.preventDefault()`. Otherwise cmd/ctrl-click-to-new-tab breaks.
- Sheet MUST be `modal={false}` per Radix. Modal mode traps focus + steals tab; we want the chat input still usable.
- ACL is enforced server-side in `api.docs.read`. For `scope='mine'` docs the caller doesn't own, the viewer renders "Not found / access denied" branch.
- `/docs/<docId>` route stays — it is the shareable URL + right-click escape hatch + fallback when the sheet provider isn't mounted (e.g. print view, deep link from email).
- System prompt does NOT need to change. Agent keeps emitting `[<docId§section>](/docs/<docId>)` citation markdown — the click handler intercepts client-side; the markdown is also a valid deep link.
- Mobile: shadcn Sheet defaults to full-width on small viewports; correct behavior, no override.

## MUST
- Defend the citation parser, not the prompt: `DOC_HREF_RE` in `packages/react/src/components/citation-anchor.tsx` stops at `§ % # ?` and consumes an optional `(?:%C2%A7|§)` separator extracting `sectionInPath`. Why: agent appends `§section` to the URL `[…](/docs/kx7…§4)` which the renderer URL-encodes to `%C2%A7`.
- Validate the captured docId against the Convex id charset (lowercase alphanumeric) via `DOC_ID_RE`, else fall through to a plain `<a>` with no Convex query. Why: a polluted docId throws `ArgumentValidationError` in `v.id("docs")`.

## NEVER
- Pass an unvalidated docId to a Convex `v.id("docs")` query. Cost: `ArgumentValidationError` lets the React error boundary swallow the entire assistant message → blank reply.

## Pitfall
- System prompt renders citations `[<docId§section>](/docs/<docId>)` — chip text carries `§section`, link target is docId only; the agent occasionally violates this, so the parser is the durable defense.
