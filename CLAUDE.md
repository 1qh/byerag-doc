# CLAUDE.md

## Recovery trigger — the founder's `resume` word

If founder types **`resume`** in any message (case-insensitive, with or without punctuation), drop everything in-flight and:

1. Run full session-start protocol below — read every doc in mandatory order
2. Read `ledger.jsonl` last entry → take `next` field as action
3. Execute that action immediately — no confirmation, no "should I?", no "ok?"

`resume` overrides any in-flight clarifying question. One word. One trigger. Always works.

## Autonomous build mode — post-compaction stance

byerag is in autonomous build mode. Founder authorized end-to-end execution from current ledger position through the final phase in `ROADMAP.md`. Compaction does NOT reset that authorization.

**Banned shapes after compaction / fresh session**:

- "What would you like me to do?" — answer is in the ledger.
- "Ready to start?" / "Should I proceed?" / "Ok to continue?" — never. Resume per ledger.
- "Want me to check X first?" — read-only checks are agent's job per `book/HARD-RULES.md` self-check-first.
- "Should I write the next ADR / table / tool?" — yes if ledger says so; never ask.
- "Plan looks good?" — plan is in this repo; if new info changes the plan, update the doc and proceed.
- "I've finished X, what next?" — ledger has next. Read it.
- Any yes/no question whose answer is implied by ROADMAP + ledger + ADR stack.

**Required posture**:

- Read everything in "Session start protocol" below first. No shortcuts.
- Read `ledger.jsonl` last entry: tells you what was just completed and what's next.
- If ledger empty / corrupt / stale (commit SHA mismatches HEAD): re-derive next action from `ROADMAP.md` phase progress + filesystem inventory, then proceed.
- Execute next concrete action without asking. Append a ledger row on completion.
- Stop only for the three valid reasons in `book/HARD-RULES.md`: credential agent cannot acquire, shared-blast-radius irreversible op, genuine MCQ fork between equivalent paths under the rule stack.
- "I might have missed something" is NOT a stop reason — doc set is comprehensive; gaps surface as concrete questions during build.

The single message after session-start protocol is INFORMATIONAL ONLY in autonomous mode — summarizes state and announces the next action being executed. Does NOT ask permission. Founder steers by interrupting if needed.

## Session start protocol

Run before any work. In exact order:

1. Read these docs in `~/tc/book/` **root files only** IN FULL (no `limit`/`offset` — entire file loads):
   - `PHILOSOPHY` · `HARD-RULES` · `NON-GOALS` · `SUBSTRATE` · `CLAUDE`
   These are the load-bearing mindset + rule set. Subfolders under `~/tc/book/` are project-specific to other project families — NEVER read at byerag session start.

2. Read every doc at this repo's root + every ADR under `docs/adr/` IN FULL. No partial reads. Each Read call omits `limit`.

3. Read every memory file at agent's persistent memory directory for this project IN FULL (MEMORY.md index + every pointer file).

4. Inspect byerag code repo filesystem: top-level entries, then `apps/admin/`, `apps/user/`, `apps/backend/`, `packages/react/`, `packages/cli/`. Confirm presence against this repo's `STACK.md` + `SCHEMAS.md`.

5. Consult this repo's `ledger.jsonl` for prior-session decisions and audit outcomes at HEAD.

## Read full, never partial

Every required doc loaded with `Read` tool, NO `limit`/`offset` args. Full file or nothing. Partial reads (head, tail, skim, paginated chunk, "first N lines for proof") banned — silent skip is the failure mode this rule kills.

No manifest theatre. No first-N-words prefix. Founder spot-checks by asking for verbatim text from random section of random doc. Wrong / missing / paraphrased = doc was not actually loaded full → restart protocol.

After full read, send a single INFORMATIONAL message (NOT a question):

- File counts read per source
- Mindset restatement in 3-5 sentences covering `book` mindset + this repo's product thesis
- Current state: which REQUIREMENTS settled, which open, which ADRs exist, which Phase from `ROADMAP.md` is next
- Ledger state: last completed action + next concrete action
- "Resuming now: <next action>"

Then execute next action without further confirmation. Re-running a green-at-HEAD ledger entry without reason violates `book/HARD-RULES.md` "Ledger consultation on session start".

## Ledger format

Append-only JSONL at `ledger.jsonl` (gitignored). Each row:

```
{"at": "<iso8601>", "commit": "<byerag sha>", "phase": "P0|P1|...", "completed": "<what just landed>", "next": "<concrete next action>", "notes": "<optional gotchas captured>"}
```

Append a row after every meaningful checkpoint (commit, decision, audit conclusion, gotcha discovery). The LAST row's `next` field is the resume point on next session start.

Read the file with `tail -1 ledger.jsonl | jq` (or read full for history).

If file missing / empty: this is the very first session OR ledger was reset. Derive `next` from `ROADMAP.md` phase progress against byerag filesystem.

If last row's `commit` mismatches current byerag HEAD: ledger stale (work happened outside agent). Re-verify state by reading current schema + tools, append a fresh row with `commit` set to current HEAD before proceeding.

## Writing discipline — canonical state, no legacy framing

byerag is brand-new. Specs (REQUIREMENTS, SCHEMAS, ROADMAP, ADRs, etc.) describe **canonical desired state** in present tense. Reader (next-session agent, founder) reads the spec and gets the design — not a transition story.

**Banned framings** in any spec doc:

- "We used to have X, now we have Y" → just describe Y.
- "Migrate from A to B" / "deprecate A" → A doesn't exist in the spec. B exists.
- "Drop / fold / merge X into Y" → Y is the SoT, carries fields a/b/c. X never appears.
- "Audit if legacy field still lingers" → if it's not in the spec, it doesn't exist; if impl has a vestige, that's an impl-vs-spec drift, fix the impl.
- "What legacy had vs what we have now" comparisons.
- "Backward-compat" / "preserve old behavior" — pre-launch, no users to be backward-compat with.

Per `book/PHILOSOPHY.md` "Unlimited rework pre-launch — throw away code freely. Better outcome > preserving past decisions" + `book/HARD-RULES.md` "Pre-launch: one migration only; reset on schema change".

**Spotting a slip in your own writing**: if you write "fold X into Y" / "drop X" / "merge X into Y" / "previously" / "formerly" in a spec doc, you're framing as transition. Rewrite as direct description of Y. X never existed in the spec.

**Implementation gap from spec is a separate concern**: if running code has fields/tables/tools the spec doesn't acknowledge, that's impl-vs-spec drift per `book/HARD-RULES.md` "Spec-of-code drift bound by tooling". Spec is canonical; fix the impl. Don't pollute spec with vestige acknowledgements.

## Confidentiality scope

byerag-docs is private. Product-specific concepts (internal docs platform, admin doc upload, user doc upload, byerag-specific tools, role-by-app pattern) live ONLY here.

byerag code repo stays generic-engineering-only. No product domain leak in commits, source, README, lint output, comments, help text. Per `book/SUBSTRATE.md` confidentiality discipline.

Banned in byerag code repo / commits / source / README / lint output: "internal docs platform", "doc Q&A", "knowledge base assistant", "admin app vs user app" framing, any product narrative. Code is shaped generically; product story lives only in byerag-docs.

## No hallucination

If a fact is not present verbatim in this repo, `~/tc/book/` root, the byerag code repo source, or `~/poc/claude2b/` reference source — you do not know it. Read more or ask. Inventing names, paths, ADR titles is forbidden.

## Commit messages

Per `book/CLAUDE.md` rules. Conventional commits. No AI / Claude / coauthor mention. Body only when "why" not obvious. Commit incrementally and silently.

## Doc evolution

Every milestone (decision landed, ADR resolved, build green, gotcha hit, scope clarified) → update owner doc + append ledger row + commit doc with the work that taught it. Never duplicate across docs. One fact, one home. Gotchas → `GOTCHAS.md` section that owns the topic, not append-only bucket.
