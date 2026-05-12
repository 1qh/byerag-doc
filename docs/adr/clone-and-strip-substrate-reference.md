# clone-and-strip-substrate-reference

Bootstrap byerag by cloning the substrate reference (a working Convex + agent-SDK + sandbox + proxy stack from another project family), stripping every product-domain layer, then layering byerag's product on top via P1+ phases.

## Beats

- **Greenfield rewrite**: substrate has working stream pipeline, proxy with cost controls, per-chat bearer scheme, CLI dispatch framework, registry codegen, React chat hooks. Rebuilding from blank takes weeks; substrate has earned its shape over many gotchas.
- **Hard fork (keep history)**: substrate is actively developed by a different project; no upstream rebase path exists. History inheritance buys nothing.
- **Selective file copy**: misses cross-file invariants (proxy ↔ schema, agent script ↔ sandbox client). Hard to keep coherent.

## Real cost

- One-time strip pass: deletes the other project's app dirs, tool providers, schema tables, embedded skill blobs, branded README.
- Cannot pull future upstream changes without manual cherry-pick.
- Inherits substrate's lint baseline + conventions (pm4ai), which is desired anyway.

## Gotcha for Claude

The substrate's `agentScript.ts` is a single embedded TS string — escape sequences inside are part of the agent script source, not the host file. Editing it requires regex-aware tooling, not naive find-replace. The `AGENT_SKILLS_BY_APP` object literal lives inside that string; for baseline, set to `{"admin":{},"user":{}}` and let per-app skills get re-injected when real tools land.

The substrate's `chats.app` field was an optional string discriminator across many app values; byerag tightens it to `'admin' | 'user'` union. Implementation has to migrate any stale rows (none at baseline because byerag's Convex backend is fresh).

The substrate's `sandboxClient.ts` ships with E2B SDK calls; byerag's P1 phase swaps in a Docker+gVisor implementation behind the same `createSandbox` / `connectSandbox` interface so the rest of the stack doesn't move.
