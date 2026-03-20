# EPUB Translator

> 🇰🇷 [한국어 버전 보기](./README.ko.md)

An AI-powered EPUB translation tool that produces fluent, context-aware translations while faithfully preserving every structural and formatting detail of the original e-book.

---

## Demo

**Side-by-side comparison — *The Linux Command Line* (EN ↔ KO)**

https://github.com/user-attachments/assets/fbfd4a3b-6cd6-47c8-8aec-3006106d7a4d

- Shell commands / code blocks and their output indentation and formatting preserved as-is
- Body text translated naturally and contextually in Korean

---

## Background

EPUB is an XML-based e-book format where content is tightly interleaved with markup. Every paragraph contains inline elements — emphasis, links, annotations, special characters — that are inseparable from the text.

This makes EPUB translation meaningfully harder than translating plain text. A naive approach of feeding the raw document to an LLM risks structural corruption: tags get rewritten, reordered, or dropped, leaving the rendered e-book broken even when the translation itself is accurate.

The core goal of this project is to translate EPUB content with no structural loss — every tag, link, and element ends up exactly where it was in the original.

---

## What It Does

- Upload an EPUB, choose source and target languages, and receive a fully translated EPUB as output
- The translated file is a valid EPUB: same chapter structure, same formatting, same links — just in a different language
- Supports long-form documents (novels, textbooks) with resumable, checkpointed jobs
- Includes a built-in web UI for monitoring progress

---

## Pipeline

```
EPUB Input
    │
    ▼
[Stage 1] Extraction
    - Walk the EPUB spine and identify all translatable content
    - Separate text from markup so only clean sentences reach the LLM
    - Skip non-translatable regions (code, math, SVG, etc.) entirely
    │
    ▼
[Stage 2] Context Building
    - Analyze the full document before any translation begins
    - Extract a domain glossary to ensure consistent term translation
    - Build per-section summaries for translation context
    - Resolve conflicting term translations across sections by frequency
    │
    ▼
[Stage 3] Translation
    - Translate in batches, passing glossary and summary as context
    - Structured output enforces a strict 1:1 mapping between
      input paragraphs and their translations
    │
    ▼
[Stage 4] Insertion
    - Reattach translated text to its original markup
    - Patch each chapter document in-place
    - Repackage into a valid EPUB archive
```

---

## What Gets Preserved

| Element | Preserved |
|---|---|
| Chapter structure and reading order | ✅ |
| Inline formatting (emphasis, links, spans, etc.) | ✅ |
| Complex embedded elements (ruby, math, SVG) | ✅ |
| Hyperlinks and cross-references | ✅ |
| Code blocks and untranslatable regions | ✅ (passed through unchanged) |
| Domain term consistency | ✅ (via pre-translation glossary) |

---

## Technical Challenges

### 1. Keeping inline markup intact through translation

EPUB paragraphs are full of inline tags mixed into the prose — links, emphasis, annotations, and more. LLMs tend to rewrite or rearrange markup when it appears in the prompt, so even an accurate translation can leave the document structurally broken. We designed a clean separation between what the LLM sees and what the final document contains, with a reliable mechanism to reunite the two afterward.

### 2. Knowing what not to translate

Not everything in an XHTML document is prose. Code blocks, mathematical expressions, SVG diagrams, and preformatted text must be left entirely untouched — including all their descendants. We implemented a filtering layer that identifies these regions and propagates the skip decision consistently through nested document trees.

### 3. Maintaining translation consistency across a long document

A full-length book is too large to translate in a single context window. When the document is split into batches, the same term can be translated differently in different sections, and each batch loses awareness of what came before. We designed a lightweight context layer — glossary and summaries — that travels with every batch without ballooning token costs.

### 4. Resuming interrupted jobs on long documents

Translating a novel involves hundreds of API calls. Network failures, rate limits, or manual stops are inevitable, and re-translating from scratch each time is wasteful. We implemented a checkpointing system fine-grained enough to resume mid-document without duplicating work, while keeping the persisted state small and portable.

### 5. Human-in-the-loop glossary review

For professional or precision use cases, automated glossary extraction isn't enough — a human needs to verify domain terms before translation begins. We designed a workflow that pauses mid-pipeline, surfaces extracted terms for review, accepts edits, and resumes seamlessly — without complicating the standard automated flow.

---

## Workflow Modes

**Classic mode**: Upload → translate → download. Fully automated.

**Glossary review mode**: The pipeline pauses after the context-building stage, before any translation begins. The extracted glossary is surfaced in the web UI for human review and editing. Translation resumes with the approved glossary applied to every subsequent call.

---

## Architecture

```
┌──────────────────────────────────────────┐
│             Desktop App                  │
│   pywebview window (no browser chrome)   │
└─────────────────┬────────────────────────┘
                  │  HTTP (localhost)
┌─────────────────▼────────────────────────┐
│           FastAPI Backend                │
│                                          │
│  Job management, SSE progress streaming  │
│  Glossary review API                     │
│  Download endpoint                       │
└─────────────────┬────────────────────────┘
                  │
┌─────────────────▼────────────────────────┐
│         Translation Pipeline             │
│                                          │
│  Extraction → Preprocessing              │
│  → Translation → Insertion               │
│                                          │
│  Checkpointed — resumes on interruption  │
└──────────────────────────────────────────┘
```

The frontend is a React + TypeScript SPA served as static files by FastAPI. The desktop shell wraps everything in a pywebview window for a native app experience.

---

## Web UI

**Main page** — Upload EPUBs, monitor live progress via SSE, manage settings.

**Glossary review page** — Inspect and edit every extracted term before translation begins (glossary review mode only).

**Result viewer** — Side-by-side original and translated chapter view.

---

## Tech Stack

| Category | Technology |
|---|---|
| Pipeline | Python 3.12, asyncio |
| Backend | FastAPI, uvicorn |
| EPUB Processing | lxml, zipfile |
| Translation | OpenAI GPT API |
| Frontend | React 18, TypeScript, Vite, Tailwind CSS |
| Desktop Shell | pywebview |

---

## Project Status

**In development** — 2025.10 ~ present

This repository contains documentation only. The source code is not publicly available.

---

## License

MIT
