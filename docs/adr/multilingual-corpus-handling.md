# multilingual-corpus-handling

Doc text is embedded as-is regardless of language. Nomic-embed-v2-moe handles ~100 languages natively in one shared embedding space. Per-doc language is detected (CLD3 / fasttext) and stored as `docs.lang: string?` for display + filtering, NOT for routing.

## Language detection

On ingest, after text extraction:

- Run language detector over the first 4 KB of `extractedText`.
- Store top-1 language ISO 639-1 code in `docs.lang`.
- If confidence < 0.7 OR text has multiple high-confidence languages, store `'mixed'`.

## Mixed-language docs

No special routing. Nomic-embed-v2 handles mixed-language input acceptably (training corpus included code-switched text). Per-chunk language can drift; the model abstracts away.

## Query language

Query is embedded with the same model, no per-language routing. Cross-language retrieval works (a Vietnamese query can find English docs and vice versa) because Nomic-v2 trained on cross-lingual retrieval objectives.

## OCR language hints

Tesseract OCR pass uses `-l eng+vie` by default. Other languages: extend the sandbox image's tesseract data packs. ADR `sandbox-image-and-cli-delivery.md` lists the installed languages; update there.

## Beats

- **Per-language embedding model**: routes a query to a model based on detected language; high cost (multiple models in memory), poor cross-lingual.
- **Translate to English on ingest**: lossy; LLM translation cost on every doc.
- **No language metadata**: loses display ("this doc is in Vietnamese") + filtering ("only EN docs in similar search").

## Real cost

- Language detector adds ~50ms per doc at ingest. Cheap.
- `docs.lang` filter in similar-search is optional (P5+).

## URL slugs for non-ASCII topic names

Vietnamese topic names (e.g. `Mục đích quy định`) URL-encode to unreadable percent-escapes, so dashboard topic routes carry a computed-on-the-fly ASCII slug with no schema field.

## MUST

- Slug a topic name with exactly this `slugify`: `s.normalize('NFD').replaceAll(/[̀-ͯ]/gu, '').replaceAll('đ', 'd').replaceAll('Đ', 'D').toLowerCase().replaceAll(/[^a-z0-9]+/gu, '-').replace(/^-+|-+$/gu, '')`. Why: strips combining diacritics + đ/Đ so VN names produce stable ASCII slugs.
- Resolve `<slug>` server-side via `dashboard.testDetail` doing `topicRows.find(t => slugify(t.name) === slug)`. Why: no slug column persisted; matched on read.

## Pitfall

- The same `slugify` is duplicated in `apps/admin/src/app/(standalone)/training/page.tsx` for client-side `Link` href construction; keep both copies byte-identical.

## Gotcha for Claude

- Mixed-language docs (a contract with English boilerplate + Vietnamese signatures) embed fine; expect lower similarity scores for queries that match only one half.
- OCR for non-eng/non-vie docs without the right tesseract pack → garbage text → garbage embedding. Detect by checking extracted text Unicode block distribution; flag for re-ingest after image rebuild.
- Right-to-left scripts (Arabic, Hebrew) render correctly in PDF preview only if PDF.js has the appropriate font fallback baked.
