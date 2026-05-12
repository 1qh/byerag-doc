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
