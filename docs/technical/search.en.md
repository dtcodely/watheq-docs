---
title: Advanced Search System
---

# Advanced Search System

The Watheq system embodies an enhanced search engine designed to allow users to retrieve documents strictly and swiftly, regardless of overall data volumes.

![Image suggestion: Gear wheel graphic characterizing a search engine logic or a screenshot of the hosted advanced search interface]

## Foundational Concepts

1.  **Full-Text Search (FTS):** Backed by the `PG_TRGM` (PostgreSQL Trigram) extension and specialized `GIN Index` logic for highly optimized rapid sweeping against massive text entries.
2.  **Autonomous Text Extraction (OCR):** A systematic Background Job executing simultaneously upon uploading any artifact (Word, PDF, PNG/JPG) converting its contents seamlessly into the `textContent` database field.
3.  **Parametric Advanced Filtering:** Flexible combinatorics fusing keywords alongside distinct entities (e.g., Dates, Sub-Departments, Types, Statuses).
4.  **Result Snippeting & Highlighting:** Intelligently stripping the context surrounding the detected word (Snippet) and highlighting it distinctly on the UI table.

---

## 1. Subsystems: Database Indexing

Provisioning a `GIN Index` stands as a pivotal performance milestone. The Trigram evaluates texts inherently not as isolated words but via continuous string sequences (n-grams), enabling robust Fuzzy Search and tolerating typos natively, matching Arabic syntax outstandingly well.

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX IF NOT EXISTS "Document_textContent_gin_idx" ON "Document" USING GIN ("textContent" gin_trgm_ops);
CREATE INDEX IF NOT EXISTS "Document_subject_gin_idx" ON "Document" USING GIN ("subject" gin_trgm_ops);
```

---

## 2. Text Extraction Service / OCR

When documenting a transaction, the engine dispenses a job event directly to a `BullMQ` cluster queue. Empowered Workers labor in the background decoding payload bytes via precise software libraries:

-   **Scanned Documents (PNG/JPEG):** Subordinate to `Tesseract.js` interpreting dual `ara+eng` linguistic packets.
-   **Word Bindings:** Extracted raw content efficiently mediated by `mammoth`.
-   **Standard Formats (PDF):** Relies upon `pdf-parse`.

Cumulatively, all extracted bodies merge directly onto the `textContent` attribute native to the Document table.

---

## 3. Backend Handling Mechanics

The active router controller (`document.controller.ts`) captures payload inputs. If an explicit `fullText` parameter signifies truth, the resolver appends the `textContent` property into the active hunt criteria relying on logical (OR) binding.

To formulate a fluid Search Snippet, the Backend applies runtime heuristics:
1. Triangulates the coordinate mapping of the sought word relying on standard `indexOf` mechanisms.
2. Carves a contextual sequence spanning approximately 60 raw characters padding the initial search match symmetrically.
3. Ships the newly fabricated string element (Snippet) nested alongside the primary API projection.

---

## 4. Frontend Highlighting Render

Populating the main `Documents.tsx` interface, Watheq renders the synthesized Snippet beneath document characteristics, safely enveloping the queried string internally with an (HTML `<mark>` Tag) prompting distinct yellow background focus states visually.
This visual engine is structurally synchronized with an underlying Advanced Search Panel offering explicit filters resolving complex Date hierarchies and multi-category tracking dynamically.
