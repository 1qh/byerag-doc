# text-extraction-by-mime

On upload, after scan-clean, a Convex action extracts text from the blob and stores the extracted text in `docs.extractedText`. Mime-to-extractor table:

| Mime | Extractor | Fallback |
|---|---|---|
| `application/pdf` | `pdftotext -layout -nopgbrk - -` (stdin → stdout) | `tesseract <img> -` per-page rendered via `pdftoppm` if pdftotext yields < 100 chars |
| `application/vnd.openxmlformats-officedocument.wordprocessingml.document` (docx) | `pandoc -f docx -t plain` | — |
| `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` (xlsx) | `python3 -m openpyxl_to_text` (small helper) or `pandoc` | — |
| `application/vnd.openxmlformats-officedocument.presentationml.presentation` (pptx) | `pandoc -f pptx -t plain` | — |
| `text/markdown`, `text/plain`, `text/html`, `application/json`, `application/xml` | raw decode (utf-8, latin-1 fallback) | — |
| `text/x-*` (code files: ts, py, go, rs, etc.) | raw decode | — |
| `image/png`, `image/jpeg`, `image/webp`, `image/tiff` | `tesseract <img> - -l eng+vie` | — |
| `application/epub+zip` | `pandoc -f epub -t plain` | — |
| `application/rtf` | `pandoc -f rtf -t plain` | — |
| anything else | reject at upload time | — |

Extraction runs inside the sandbox image (which carries all the binaries), invoked by a Convex action via `sandbox.commands.run` on a short-lived extraction container. Result text written back to `docs.extractedText`. Embedding then runs on the extracted text.

## Beats

- **LLM extraction for everything**: unbounded cost; the long tail (scanned PDFs) is the only place LLM helps over deterministic tools.
- **Single tool (pandoc) for all formats**: pandoc handles docx/pptx/epub/rtf well but is wrong for PDFs and useless for images.
- **Client-side extraction in browser**: shifts CPU + adds upload-time latency; users on slow machines suffer.

## Real cost

- Extraction time per doc: <1s for text, 2-5s for PDF, 5-30s for OCR (per page count).
- OCR storage: extractedText for OCR'd PDFs can be noisy; threshold + manual cleanup later.
- Sandbox cold-start tax on extraction (mitigated: per-owner sandbox reused, extraction happens in same container).

## Gotcha for Claude

- `pdftotext` returns near-empty for image-only PDFs; check `len(text) < 100` post-extract → trigger OCR fallback.
- Tesseract is slow on CPU; cap PDF page count for OCR at 50 pages, warn user on overrun.
- `pandoc` strips images by default; that's correct — embeddings work on text only. Image-only PDFs go through OCR path.
- Code files (`*.ts`, `*.py`, etc.) embed as raw text; agent's `docs grep` is the right tool for them, not `docs similar`.
- Long docs are chunked AT QUERY TIME for `docs similar` (per `embedding-chunking-strategy.md`); extracted text is stored whole.
