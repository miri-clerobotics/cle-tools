# PDF Compressor Design

## Context

CLE Robotics users sometimes need to share confidential external-facing PDF materials without using a cloud upload service. The first release will add a local-only PDF size reduction tool to the existing internal tools portal.

The tool must run in the browser, keep the original file untouched, and produce a downloadable compressed copy. The first scope is PDF only. PPT/PPTX and other document formats are intentionally deferred.

## Goals

- Add a `document-compressor/` static web tool and link it from the main portal.
- Process files locally in the browser with no server upload.
- Compress PDFs by regenerating each page as a JPEG-backed PDF.
- Provide simple presets for sharing-oriented output.
- Show original size, output size, savings percentage, page count, progress, and elapsed processing time.
- Warn clearly that the first version creates a sharing copy and may remove searchable text, links, forms, annotations, and accessibility metadata.
- Preserve the original input file and download only a new output file named from the source file.

## Non-Goals

- PPT/PPTX compression.
- Server-side compression.
- Guaranteed compression below a specific target size.
- Lossless structure-preserving PDF optimization in the first release.
- Editing, merging, splitting, signing, OCR, or password removal.

## Research Summary

Three implementation families were considered.

### PDF.js + jsPDF Page Regeneration

This approach renders each PDF page to a canvas with PDF.js, exports the canvas as JPEG, and creates a new PDF with jsPDF. It is the recommended MVP approach because it fits the current static HTML repository, works fully in-browser, and has low licensing risk.

Tradeoff: it rasterizes pages. The output is good for external sharing and visual review, but it will generally lose selectable text, text search, links, forms, comments, embedded metadata, and accessibility structure.

Sources:

- PDF.js examples: https://mozilla.github.io/pdf.js/examples/
- jsPDF `addImage` compression option: https://artskydj.github.io/jsPDF/docs/module-addImage.html
- PDF.js license: https://mozilla.github.io/pdf.js/

### Ghostscript WASM

Ghostscript can perform more traditional PDF rewriting with `pdfwrite` and preset settings such as `/screen` or `/ebook`. This can preserve more PDF structure than page rasterization and may compress many PDFs well.

Tradeoff: Ghostscript has heavier WASM/runtime cost and is dual-licensed under AGPL or commercial terms. That needs internal approval before bundling or distributing it in the company tool.

Sources:

- Ghostscript WASM demo: https://github.com/laurentmmeyer/ghostscript-pdf-compress.wasm
- Ghostscript `pdfwrite` docs: https://ghostscript.readthedocs.io/en/latest/VectorDevices.html
- Ghostscript licensing: https://ghostscript.com/releases/gsdnld.html

### qpdf WASM / Structure Optimization

qpdf is useful for content-preserving PDF transformations, object stream compression, linearization, splitting, merging, and inspection. Its Apache 2.0 license is attractive.

Tradeoff: qpdf generally does not solve the common "large image-heavy PDF" case as aggressively as image downsampling/recompression. It is better as a future "preserve text and links" mode than the initial high-impact compressor.

Sources:

- qpdf purpose discussion: https://github.com/qpdf/qpdf-dev/discussions/1
- qpdf license: https://qpdf.readthedocs.io/en/stable/license.html
- qpdf WASM package: https://github.com/neslinesli93/qpdf-wasm

## Recommended First Release

Build a browser-only PDF compressor with PDF.js and jsPDF.

The tool will be framed as a "sharing copy" generator:

- Best for image-heavy PDFs and presentation-style documents.
- Good when external recipients only need to view the document.
- Not appropriate when text selection, links, form fields, comments, signatures, or accessibility semantics must be preserved.

## User Experience

The page will use the existing CLE tool visual language: light work surface, left control panel, restrained red accent, compact operational layout, and clear result area.

Primary flow:

1. User opens "PDF 압축기" from the portal.
2. User drops or selects one or more PDF files.
3. User chooses a preset:
   - `작게`: lower render scale and lower JPEG quality.
   - `균형`: default sharing setting.
   - `선명하게`: larger output, higher visual fidelity.
4. Optional advanced controls can adjust render scale and JPEG quality directly.
5. User clicks compress.
6. Tool processes pages sequentially and displays progress.
7. Tool shows original size, compressed size, savings, page count, and elapsed time.
8. User downloads the compressed PDF.

Required UI warnings:

- "파일은 외부 서버로 업로드되지 않습니다."
- "공유용 PDF로 다시 생성되며 텍스트 검색, 링크, 주석, 양식이 사라질 수 있습니다."
- "원본 파일은 수정하지 않습니다."

## Compression Presets

Initial preset values may be tuned during implementation:

- `작게`: render scale around `1.0`, JPEG quality around `0.55`.
- `균형`: render scale around `1.4`, JPEG quality around `0.72`.
- `선명하게`: render scale around `1.8`, JPEG quality around `0.86`.

The UI should label these by outcome, not technical terms. Advanced controls may expose the technical values for users who need control.

## Architecture

### Files

- `document-compressor/index.html`: complete static tool page.
- `index.html`: add portal link.
- `README.md`: add tool description and file listing.

### Client Modules

The first implementation can live in a single HTML file to match existing tools. Keep the script organized by function:

- File intake and validation.
- PDF loading and page rendering.
- PDF output generation.
- Progress/result state.
- Download helpers.
- UI event binding.

If the file grows too large or future modes are added, split into dedicated JS assets in a later refactor.

### Processing Flow

1. Read the selected file with `File.arrayBuffer()`.
2. Load the PDF with `pdfjsLib.getDocument({ data })`.
3. For each page:
   - Fetch page.
   - Create viewport with the selected render scale.
   - Render page to an offscreen canvas.
   - Convert canvas to JPEG blob/data URL using selected quality.
   - Add a new jsPDF page matching the original page aspect ratio.
   - Place the JPEG to fill the page.
   - Release canvas memory after each page.
4. Save the jsPDF document to a blob.
5. Compare output size to input size and show result.

## Error Handling

Show actionable messages for:

- Non-PDF files.
- Empty files.
- Encrypted/password-protected PDFs that cannot be opened.
- Damaged PDFs.
- Browser memory failure or canvas size limits.
- Output larger than input.

If output is larger than input, still allow download but mark it clearly and suggest a smaller preset.

## Performance

- Process pages sequentially to limit memory use.
- Reset canvas width/height after each page to release backing storage.
- Display current page progress.
- Add a cancel button if implementation complexity stays reasonable.
- Avoid loading every rendered page into memory at once.

## Privacy and Security

- No file upload.
- No remote API calls for user documents.
- Prefer vendored local library files over CDN if feasible in this repository. If CDN is used initially, document that the PDF bytes are not sent to the CDN, but library code is fetched from it.
- Do not persist files to local storage or IndexedDB.
- Clear in-memory references when the user removes files or reloads the page.

## Future Enhancements

- Batch PDF compression with per-file results.
- Target size mode that iteratively lowers quality until below a requested size or a minimum quality threshold.
- qpdf-based "preserve text and links" mode.
- Ghostscript WASM mode if licensing is approved.
- Side-by-side preview of original and compressed first page.
- Optional ZIP download for batch output.
- Drag-to-reorder or merge is out of scope unless requested later.

## Testing Plan

Manual browser verification:

- Open the portal and confirm the new tool link works.
- Load a normal PDF and compress with each preset.
- Confirm output downloads and opens.
- Confirm progress and result metrics are accurate.
- Confirm original file is not modified.
- Confirm a non-PDF file is rejected.
- Confirm an encrypted or invalid PDF shows an understandable error.
- Test at desktop and mobile widths for layout integrity.

Automated tests are limited because this repository is static HTML without a test harness. If a browser automation setup is added, cover file validation, preset state, and result rendering with Playwright.
