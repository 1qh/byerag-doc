# OPEN-QUESTIONS

Pending MCQ-shaped forks the rule stack genuinely cannot settle. Each entry has options + recommendation + reasoning. Resolve by appending a new ADR; remove the entry here when resolved.

## OQ-001 — Embedding dim policy

**Question**: store embeddings at full 768 dim, or downcast to 256 via Matryoshka at ingest?

**Options**:

- A. Store at 768. Query at 768 by default; client can request 256/512 for cheaper recall.
- B. Store at 256 always. Lose top-end recall; save 3× storage + faster ANN.
- C. Store both 768 and 256 in two fields. Pay 1.3× storage; per-query choose.

**Recommendation**: A. Storage at internal-team scale is negligible; recall quality matters more. Per `book/PHILOSOPHY.md` "Complexity-tolerant for UX". Revisit if corpus exceeds ~1M docs.

**Reasoning**: 768 fp64 × 1M docs ≈ 6 GB. Trivial on the self-host box.

## OQ-002 — Text extraction toolchain inside sandbox

**Question**: `pdftotext` (poppler) vs `pandoc` vs LLM extract for non-trivial PDFs (scanned, tables, multicolumn)?

**Options**:

- A. `pdftotext` only. Cheap, works for 80% of PDFs.
- B. `pdftotext` first; if low text yield, fall back to OCR (`tesseract`).
- C. LLM extract via Kimi (counts against budget).

**Recommendation**: B. OCR fallback is cheap on CPU; LLM extract is unbounded cost.

**Reasoning**: most internal docs are text-layer PDFs. Scanned exceptions warrant OCR; LLM extract would burn budget on the long tail.

## OQ-003 — Chat history retention

**Question**: how long do we keep `messages` + `streamEvents` rows?

**Options**:

- A. Forever. Storage cheap; useful for retrospective.
- B. 90 days, auto-purge via Convex cron.
- C. Per-user opt-in retention setting; default 30 days.

**Recommendation**: A for P0, revisit when storage grows past 10 GB.

**Reasoning**: internal team scale; chat content is the audit trail.

## OQ-004 — Sandbox image base

**Question**: which base image for the sandbox container?

**Options**:

- A. `node:20-slim` + apt install ripgrep + poppler + pandoc.
- B. Custom Debian slim w/ everything pre-baked.
- C. Distroless w/ bun + only the tools we explicitly add.

**Recommendation**: A. Familiar, debuggable, fast first-build.

**Reasoning**: image size doesn't matter at one-machine scale; debuggability does.
