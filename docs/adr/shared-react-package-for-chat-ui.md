# shared-react-package-for-chat-ui

All chat plumbing — hooks, stream parsers, components, registries — lives in `packages/react`. Apps consume via thin imports + a few config props.

## Beats

- **Per-app duplication**: chat plumbing drifts across apps; bug fixes touch N apps.
- **Component library published to npm**: external publication overhead (versioning, release notes) without an external consumer. Premature.
- **Server-side rendering of chat UI**: chat is fundamentally client-side (streaming + WebSocket); SSR doesn't buy anything.

## Real cost

- `packages/react` is consumed by both apps; breaking changes require coordinating both. Lockstep policy.
- Workspace dependency mapping (bun workspaces); standard pattern, no extra cost.

## Gotcha for Claude

- The substrate reference's `packages/react` carries `useStreamingChat` hook + chat shell components + tool-call renderers + citation chip + doc panel — all generic.
- Per-app customization lands as registry props on the chat shell: `messageRegistries`, `toolRegistries`, `sidebarSections`. Apps register their domain-specific components into these registries without forking the shell.
- No barrel `index.ts` in app code per pm4ai convention; `packages/react/src/index.ts` is the public barrel, app code imports from there.
- Component look: shadcn as-is, semantic colors, `cn()` for conditional classes, minimal DOM.

## MUST
- Wrap children in `PaneProvider` inside `DefaultProviders`, order `TooltipProvider > PaneProvider > VerbosityProvider` in `packages/react/src/next/default-providers.tsx`. Why: `message-text-part.tsx` calls `usePane()`, else throws on first message render.
- Use the base-ui `render` prop, not `asChild`; `DropdownMenuTrigger`/`AlertDialog` use `render={<X/>}`. Why: shadcn here is base-ui, not Radix.
- Render a link-styled button as `<Link className={cn(buttonVariants({variant}))}>`. Why: base-ui Button `render`'d as non-`<button>` warns `nativeButton`.
- Depend only on `react-pdf`; resolve worker via `new URL('pdfjs-dist/build/pdf.worker.min.mjs', import.meta.url)`. Why: react-pdf ships its own `pdfjs-dist`; a separate pin diverges main/worker versions.
- Wrap the pdf consumer in `dynamic(() => import('./pdf-preview'), { ssr: false })`. Why: pdf.js touches `DOMMatrix` at module top, crashes Next 16 SSR.
- Concatenate consecutive `role='assistant'` chunks via `mergeAssistantRuns` in `chunksToMessages` (`packages/react/src/lib/ui-messages.ts`) into one synthetic `UIMessage` before render. Why: agent emits one chunk per reasoning step/tool call as a separate row → wall of cards otherwise.
- Treat `reasoning|status|data-tool-x|text` as activity in `PurePreviewMessage` (`packages/react/src/components/message.tsx`); final answer is LAST `text` part whose index > `lastActivityIndex(parts)`. Why: buffers all interim activity into one `ThinkingBlock`, final text renders outside.
- Compute per-block elapsed from `startedAt` = first render (`useState(() => Date.now())`) to `isLoading` false. Why: reload of a finished message fires no transition → omits "for Ns", shows `Thought · tool1`.

## NEVER
- Use `<Button asChild>`. Cost: leaks `aschild` to the DOM (React unknown-prop error) and does not compose.
- Pin `pdfjs-dist` as an explicit dep in `packages/react/package.json`. Cost: worker diverges from react-pdf's bundled main thread — `API version 5.4.296 does not match Worker version 6.0.227`.

## Pitfall
- Streaming → `ThinkingBlock` pill auto-expands "Thinking…" + spinner; on `isLoading` false collapses to `Thought for Ns · tool1, tool2, …` with click-to-expand chevron. Inline tool blocks inside the pill use light `border border-muted-foreground/15 bg-background/60` chrome (no `Tool` card header) to nest naturally.
- Iframe-based fallback for plain blob URLs works in headed Chromium but not headless; in-app preview canvas-rendered via `<Document>`/`<Page>` is the legitimate path.
