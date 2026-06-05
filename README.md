# byerag-doc

Plans + decisions + protocol for the byerag product. Code lives in a sibling repo.

```mermaid
mindmap
  root((byerag-doc))
    Protocol
      CLAUDE — session-start + autonomous mode + ledger format
    Thesis
      VISION — what byerag is
      USERS — admin and user roles
      NON-GOALS — scope defenses
    Specs
      REQUIREMENTS — feature catalog
      ROADMAP — phase sequence
      STACK — locked stack
      SCHEMAS — convex tables
      AUTH — google oauth, role on user account
      AGENT-DOCTRINE — agent loop and prompts
      CLI-SURFACE — docs CLI commands agent sees
      UX-DOCTRINE — admin app and user app shape
      SECURITY — egress, sandbox, proxy, keys
    Audit
      VERIFY — end-state checklist
      GOTCHAS — evolving per-topic gotchas
      OPEN-QUESTIONS — pending MCQs
    Decisions
      docs/adr — one file per decision
    Audit trail
      ledger.jsonl — append-only outcome log
```

Read order at session start lives in `CLAUDE.md`. Generic engineering rules inherited from `~/book/` root.
