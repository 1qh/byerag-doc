# session-start-book-root-only

At session start, the agent reads `~/book/` root files only: `PHILOSOPHY`, `HARD-RULES`, `NON-GOALS`, `SUBSTRATE`, `CLAUDE`. Subfolders under `~/book/` (`kotlin-swift/`, etc.) are project-specific to other project families and are NEVER read at byerag session start.

## Beats

- **Read everything in `~/book/`**: includes unrelated project ADRs (Kotlin/Swift stack, iOS deploy, etc.). Pure noise; dilutes signal of rules that govern byerag.
- **Read nothing from `~/book/`**: loses the generic engineering bible.
- **Per-session opt-in (agent picks which book files to read)**: drifts; agent forgets a rule and re-discovers it.

## Real cost

- Project-specific generic rules that emerge during byerag work must land in `byerag-doc/` (or be lifted to `book` root if they apply portably). Discipline cost.

## Gotcha for Claude

- The book has a folder for each project family (`kotlin-swift/`, etc.). Reading them at byerag session start is a violation.
- If a byerag-specific rule shaped during work is portable across projects, propose it to the operator for inclusion in `book` root. Otherwise it stays in `byerag-doc/`.
- The session-start file count message reports `book root: 5 files` (not "book: 50 files") — clarity that the agent read the right slice.
