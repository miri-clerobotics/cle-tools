# PDF Compressor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a browser-only PDF compression tool for confidential sharing copies and make it manually testable from a local dev server.

**Architecture:** Add a new static `document-compressor/` page linked from the existing portal. The page uses PDF.js to render each PDF page to canvas and jsPDF to rebuild a JPEG-backed sharing PDF, with clear privacy and structure-loss warnings.

**Tech Stack:** Static HTML/CSS/JavaScript, PDF.js browser build, jsPDF UMD build, local PowerShell dev server for manual testing.

---

### Task 1: Add Portal and Documentation Entries

**Files:**
- Modify: `index.html`
- Modify: `README.md`

- [ ] Add a portal button linking to `/document-compressor/` with the label `문서 용량 줄이기`.
- [ ] Add README section explaining PDF compression, local-only processing, and the generated sharing-copy limitation.
- [ ] Verify the portal still renders as static HTML.

### Task 2: Build Static PDF Compressor Page

**Files:**
- Create: `document-compressor/index.html`

- [ ] Create the tool layout with header, upload zone, preset controls, advanced controls, processing status, result list, and manual-test friendly messaging.
- [ ] Add file validation for PDFs only, empty files, and per-file state.
- [ ] Implement PDF.js loading, page-by-page canvas rendering, jsPDF output generation, progress updates, cancellation guard, and result download.
- [ ] Release canvas memory after every page.
- [ ] Show privacy warning, structure-loss warning, original size, output size, savings, page count, elapsed time, and output-larger-than-input guidance.

### Task 3: Verify and Prepare Local Manual Testing

**Files:**
- No code files expected.

- [ ] Run static checks for expected files and links.
- [ ] Start a local static file server from the worktree.
- [ ] Confirm the local URL for manual testing.
- [ ] Commit implementation changes after verification.
