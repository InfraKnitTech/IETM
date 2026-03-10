# IETM Management System — Project Status

**Last Updated**: March 10, 2026
**Total Lines Written**: ~22,000+ (across 70+ source files + tests)
**Phase 1 Test Status**: 26/26 passing
**Phase 2 Status**: All 46 files implemented, server running, APIs tested end-to-end
**Phase 3 Status**: React frontend complete — all pages, XML viewer, fleet images
**PDF Intelligence Pipeline v2**: Complete — 8-step extraction, interactive diagram hotspots
**Interactive Hotspots**: Complete — red numbered circles on PDF pages, floating definition panels

---

## What Is This Project?

A **production-ready, secure, offline (air-gapped) web-based Interactive Electronic Technical Manual (IETM) Management System** for defense/government environments. Compliant with **S1000D, JSG0852, MIL-STD-3031, ATA iSpec 2200** standards. Designed for 1000 concurrent users with zero internet dependency.

**Core Innovations**:
1. An AI-powered Content Intelligence Pipeline that automatically classifies document sections into a military taxonomy
2. A full PDF Intelligence Pipeline that extracts diagrams, detects numbered callout labels via OCR, maps them to component names from label tables, and overlays interactive hotspot buttons directly on the PDF viewer — click a numbered circle and instantly see the component name with an LLM-generated definition

---

## Overall Progress

| Phase | Description | Status |
|-------|-------------|--------|
| **Phase 1** | Content Intelligence Pipeline (AI + classification) | COMPLETE |
| **Phase 2** | Backend Platform (FastAPI, DB, Auth, APIs) | COMPLETE |
| **Phase 3** | Frontend (React, XML viewer, fleet images) | COMPLETE |
| **PDF Intelligence v2** | 8-step extraction pipeline + interactive hotspots | COMPLETE |
| **Phase 5** | Search System (FTS + semantic + hybrid) | NOT STARTED |
| **Phase 6** | Content Authoring & Admin (editor, dashboard) | NOT STARTED |
| **Phase 7** | Export, Reporting & Hardening (PDF export, security) | NOT STARTED |

---

## PDF Intelligence Pipeline — Deep Technical Walkthrough

This is the crown jewel of the system. When a user uploads a PDF, an 8-step pipeline runs in the background, extracting everything from diagrams to terminology. Here is exactly how each step works under the hood:

### High-Level Pipeline Flow

```
PDF Upload → Save to disk → Extract TOC
                               │
                               ▼
              ┌─── Background asyncio Task ───┐
              │                               │
              │  Step 0: Scan Pages            │  PyMuPDF fitz.open() → iterate pages
              │          Extract Sections      │  → get_text("blocks") per page
              │                               │
              │  Step 1: Extract Diagrams      │  get_images(full=True) per page
              │          250px filter + dedup  │  → SHA256 hash → save to disk
              │          + page bounding box   │  → get_image_rects(xref) → % coords
              │                               │
              │  Step 2: Figure Captions       │  get_text("dict") → text blocks
              │                               │  → scan 150pt below each image
              │                               │  → regex: "Fig. 1-22"
              │                               │
              │  Step 3: Figure Titles         │  pdfplumber → structured tables
              │                               │  → match fig refs to component names
              │                               │
              │  Step 4: Callout Detection     │  PIL resize → 600px width
              │          (parallel, 4 threads) │  → EasyOCR digits-only
              │                               │  → ≥2 distinct numbers = callout
              │                               │
              │  Step 5: OCR Hotspots          │  PIL resize → 1200px width
              │          (parallel, 4 threads) │  → EasyOCR digits-only
              │                               │  → bbox → % coordinates
              │                               │
              │  Step 6: Label Tables          │  pdfplumber tables + regex fallback
              │                               │  + circled number ①② fallback
              │                               │  → map label→component_name
              │                               │
              │  Step 7: Terminology           │  Full text extraction
              │                               │  → frequency counting + NLP
              │                               │  → context sentence extraction
              │                               │
              └───────────────────────────────┘
                               │
                               ▼
              Document status → "processed"
              SSE progress stream → frontend updates in real-time
```

### Progress Tracking

Every step reports real-time progress via `progress_store`:
- `progress_store.create_pipeline(doc_id, step_names)` → initializes 8 steps
- Each step calls `progress_store.update_step(doc_id, step, status, current, total, detail)`
- Frontend connects via SSE: `GET /api/documents/progress/{doc_id}` → `text/event-stream`
- Frontend shows live step-by-step progress bars during processing

---

### Step 0: Scan Pages & Extract Sections

**Library**: PyMuPDF (`fitz`)
**File**: `pdf_content_extractor.py`

```
Algorithm:
1. Open PDF with fitz.open(file_path)
2. Get total page count from len(doc)
3. Flatten the TOC tree (from PDF bookmarks) into an ordered list
4. For each TOC entry:
   a. Compute page range: page_start → next_entry.page_start - 1
   b. For each page in range:
      - page.get_text("blocks") → list of text blocks
      - Each block: (x0, y0, x1, y1, text, block_no, block_type)
      - Filter: block_type == 0 (text only, skip images)
      - Skip empty blocks
   c. Join text blocks with "\n\n"
   d. Create PdfSection record: {section_title, content_text, page_start, page_end}
5. flush to DB
```

**Output**: `PdfSection` records in database with full text per section

---

### Step 1: Extract Diagrams (Image Extraction + Dedup + Bounding Box)

**Library**: PyMuPDF (`fitz`) + `hashlib`
**File**: `pdf_intelligence_service.py` → `_extract_diagrams_with_progress()`

This step finds every image in the PDF, filters out small ones, deduplicates, and computes where each image sits on the page:

```
Algorithm:
1. fitz.open(file_path)
2. For each page (0 to total_pages):
   a. page.get_images(full=True) → list of (xref, smask, width, height, ...)
      - xref = internal PDF cross-reference ID for the image object

   b. For each xref:
      - doc.extract_image(xref) → {width, height, image (raw bytes), ext (png/jpg)}

      - SIZE FILTER: Skip if width < 250 OR height < 250
        Why 250? Eliminates bullet points, small logos, header icons, decorative
        borders — only keeps substantial technical diagrams

      - HASH: hashlib.sha256(img_bytes).hexdigest()
        64-char hex string uniquely identifies identical images

      - DEDUP: Check if hash already seen within same document
        If duplicate → skip (same image repeated on different pages)
        Uses in-memory set, not cross-document DB query

      - SAVE: Write raw bytes to storage/uploads/diagrams/{doc_id}/diagram_{n}.{ext}

      - BOUNDING BOX:
        page.get_image_rects(xref) → list of fitz.Rect objects
        rect = rects[0]  (use first occurrence on page)

        Convert to page percentage coordinates (0-100):
          page_x = (rect.x0 / page.rect.width) * 100    ← left edge %
          page_y = (rect.y0 / page.rect.height) * 100   ← top edge %
          page_w = ((rect.x1 - rect.x0) / page.rect.width) * 100   ← width %
          page_h = ((rect.y1 - rect.y0) / page.rect.height) * 100  ← height %

        Why percentages? The PDF page might render at any zoom level.
        By storing as %, the frontend can position hotspot overlays
        at the correct location regardless of scale.

3. Create Diagram records in DB with all fields
```

**Output**: `Diagram` records with `image_path`, `width`, `height`, `image_hash`, `page_x/y/w/h`

---

### Step 2: Figure Caption Extraction (Proximity-Based Text Matching)

**Library**: PyMuPDF (`fitz`)
**File**: `diagram_extractor.py` → `extract_figure_captions()`

This step finds text like "Figure 1-22" near each diagram image by looking at text blocks below the image:

```
Algorithm:
1. Group diagrams by page_number
2. Regex pattern: r'(?i)fig(?:ure)?\.?\s*(\d+\s*[\-\.]\s*\d+)'
   Matches: "Fig. 1-22", "Figure 3.5", "fig 12-1", "FIGURE 7-14"

3. For each diagram on each page:
   a. Get image bounding box: page.get_image_rects(xref)
      img_bottom = rect.y1  (bottom edge of image in PDF points)

   b. Get all text blocks with positions:
      page.get_text("dict") → {"blocks": [{type, bbox, lines: [{spans: [{text}]}]}]}

   c. For each text block:
      block_top = block.bbox[1]  (top edge of text block)

      PROXIMITY CHECK: Is this text block within 150pt below the image?
        block_top >= img_bottom AND block_top <= img_bottom + 150

      Why 150pt? (~2 inches) — captions in technical manuals are typically
      directly below the figure, within 1-2 lines. 150pt catches most
      caption styles while avoiding unrelated text further down.

      If match found → Extract text from all spans in block
      → Run regex → If "Fig X-Y" found → Store as diagram.figure_name
      → Break (use first match only)

4. Update Diagram records with figure_name
```

**Output**: Updates `diagram.figure_name` (e.g., "Fig. 1-22")

---

### Step 3: Figure Title Table Extraction

**Library**: pdfplumber
**File**: `label_table_extractor.py` → `extract_figure_name_tables()`

Many technical manuals have a "List of Figures" table mapping figure numbers to titles. This step scans ALL pages for such tables:

```
Algorithm:
1. Open PDF with pdfplumber.open()
2. For EVERY page:
   a. Method 1 — Structured table extraction:
      page.extract_tables() → list of tables (list of rows)
      For each cell: Search for fig reference pattern
      Pattern: r'(?i)fig(?:ure)?\.?\s*(\d+\s*[\-\.]\s*\d+)'
      If found in one column → look at other columns for component name
      Filter: name must be > 2 chars, not all digits, not another fig ref

   b. Method 2 — Inline text fallback:
      page.extract_text() → raw page text
      Pattern: r'(?i)fig(?:ure)?\.?\s*(\d+[\-\.]\d+)\s*[.\-–—:]\s*([A-Z][A-Za-z\s\-/()]+)'
      Matches: "Figure 1-22. Micrometer" or "Fig 3-5 — Hydraulic Pump"

3. Normalize references: "1 . 22" → "1-22", "3.5" → "3-5"
4. Build map: {"1-22": "Micrometer", "3-5": "Hydraulic Pump"}
5. Match to diagrams: For each diagram with figure_name "Fig. 1-22"
   → Extract ref "1-22" → Look up title → Set diagram.figure_title = "Micrometer"
```

**Output**: Updates `diagram.figure_title` (e.g., "Micrometer")

---

### Step 4: Callout Candidate Check (Parallel Downscaled OCR)

**Library**: EasyOCR + PIL (Pillow) + numpy
**File**: `diagram_extractor.py` → `has_numeric_callouts()` + `run_callout_checks_parallel()`

This is a fast screening step — before running full OCR, we quickly check which diagrams actually have numbered callout labels. Most diagrams (photos, schematics without labels) don't have callouts, so this saves significant processing time:

```
Algorithm:
1. Pre-load EasyOCR reader singleton (lazy, first call loads the model)
   - easyocr.Reader(["en"], gpu=False)
   - Model cached at EASYOCR_MODEL_DIR
   - First load: downloads ~100MB of models (one-time)

2. ThreadPoolExecutor(max_workers=4) — check diagrams in parallel

3. Per diagram (in thread):
   a. PIL.Image.open(image_path)

   b. DOWNSCALE to 600px max width:
      ratio = 600 / img.width
      img.resize((600, new_height), LANCZOS)

      Why 600px? This is a PREVIEW check — we just need to know
      if numbers exist, not their exact positions. 600px is enough
      for OCR to detect digits while being 3-5x faster than full res.

   c. Convert to numpy array

   d. EasyOCR readtext():
      - allowlist="0123456789"  ← ONLY detect digits (faster, fewer false positives)
      - paragraph=False  ← treat each detection independently

   e. Post-process results:
      - Filter by confidence >= 0.3 (EasyOCR returns 0.0-1.0)
      - Extract digit groups: re.findall(r'\d+', text)
      - Keep only values 1-99 (callout label range)
      - Count distinct digit groups

   f. DECISION: ≥2 distinct digit groups found → has_callouts = True
      Why 2? A single number could be a page number or part number.
      Two or more distinct small numbers strongly suggests callout labels.

4. Timeout: 60 seconds per diagram (handles first-run model loading)
5. Update diagram.has_callouts in DB
```

**Output**: Updates `diagram.has_callouts` boolean. Typically 5-15% of diagrams have callouts.

---

### Step 5: OCR Hotspot Detection (Parallel, Full Resolution)

**Library**: EasyOCR + PIL + numpy
**File**: `diagram_extractor.py` → `detect_hotspots()` + `run_hotspot_detection_parallel()`

Only runs on diagrams where `has_callouts=True` (from Step 4). This does accurate OCR to find the exact position of each numbered label:

```
Algorithm:
1. Filter: Only process diagrams with has_callouts=True
   (Skips the 85-95% of diagrams without callouts)

2. ThreadPoolExecutor(max_workers=4)

3. Per callout diagram (in thread):
   a. PIL.Image.open(image_path)
   b. Record original dimensions: img_w, img_h

   c. DOWNSCALE to 1200px max width:
      ratio = 1200 / img_w
      img.resize((1200, new_h), LANCZOS)

      Why 1200px? Balances accuracy vs speed:
      - 600px (Step 4): too low for precise positioning
      - Full res (e.g. 4000px): too slow, no accuracy gain
      - 1200px: EasyOCR detects all labels accurately

      Compute scale factors:
        scale_x = original_width / 1200
        scale_y = original_height / new_height
      These are used to map OCR bbox coordinates back to original image space.

   d. Convert to numpy array

   e. EasyOCR readtext():
      - allowlist="0123456789"
      - paragraph=False
      Returns: [(bbox, text, confidence), ...]
      bbox = [[x1,y1], [x2,y2], [x3,y3], [x4,y4]]  (4 corners of text region)

   f. Post-process each detection:
      - Skip if confidence < 0.3
      - Parse as integer, skip if not 1-99
      - Skip if duplicate label_number (first occurrence wins)

      - Scale bbox back to original image coordinates:
        xs = [point[0] * scale_x for point in bbox]
        ys = [point[1] * scale_y for point in bbox]
        x_min, x_max = min(xs), max(xs)
        y_min, y_max = min(ys), max(ys)

      - Convert to PERCENTAGE of image dimensions:
        x = (x_min / img_w) * 100
        y = (y_min / img_h) * 100
        width = ((x_max - x_min) / img_w) * 100
        height = ((y_max - y_min) / img_h) * 100

        Why percentages? Same image may be rendered at different sizes.
        Percentages work at any scale.

4. Create Hotspot records: {diagram_id, label_number, x, y, width, height}
```

**Output**: `Hotspot` records in DB. Each has label_number (e.g., "5") and position as % within the diagram image.

---

### Step 6: Label Table Extraction (Component Name Mapping)

**Library**: pdfplumber
**File**: `label_table_extractor.py` → `extract_label_tables()`

Now we know WHERE the labels are (from Step 5), but we need to know WHAT they refer to. Technical manuals typically have a parts list table on the same page or next page:

```
Algorithm:
1. Collect pages to scan: For each callout diagram
   → check diagram_page AND diagram_page + 1
   (Parts lists are often on the page facing the diagram)

2. Per page, THREE extraction strategies (in order):

   Strategy 1 — Structured table extraction:
   ┌─────┬──────────────────┐
   │  1  │ Opening clamp    │  ← pdfplumber detects table structure
   │  2  │ Piston           │     cell[0] = number (int 1-99)
   │  3  │ Cylinder head    │     cell[1] = component name
   └─────┴──────────────────┘

   Strategy 2 — Regex on page text (fallback):
   Pattern: r'(\d{1,2})\s*[.\-–—]\s*([A-Za-z][\w\s,/()]+)'
   Matches: "1. Opening clamp" or "2 - Piston" or "3 — Cylinder head"
   Filter: num must be 1-99, name must be > 2 chars, name not all digits

   Strategy 3 — Circled number support (fallback):
   Handles Unicode circled numbers: ① ② ③ ... ⑳
   Mapping: CIRCLED_NUMBERS = {chr(0x2460+i): str(i+1) for i in range(20)}
   Pattern: r'([①-⑳])\s*([A-Za-z][\w\s,/()\-]+?)(?=[①-⑳]|\Z)'
   Matches: "① Opening clamp ② Piston" → splits on circled numbers
   Maps ① → "1", ② → "2", etc.

3. Deduplicate by label number (first occurrence wins)

4. Match hotspots to components:
   For each hotspot with label_number "5":
   → Look in page_labels[diagram_page]["5"]
   → If found → hotspot.component_name = "Opening clamp"
   → Also check diagram_page + 1 (parts list might be on next page)
```

**Output**: Updates `hotspot.component_name` (e.g., "Hydraulic Pump", "Opening clamp")

---

### Step 7: Terminology Extraction

**Library**: regex, collections.Counter, NLTK (stopwords)
**File**: `terminology_extractor.py`

Extracts technical vocabulary from the full document text using frequency analysis and NLP heuristics:

```
Algorithm:
1. FULL TEXT EXTRACTION:
   - Re-extract all pages: page.get_text("text") per page
   - Join into single string

2. SINGLE-WORD TERM EXTRACTION:
   - Regex: re.findall(r'\b[a-zA-Z]{4,}\b', text)
     Why 4+ chars? Filters "the", "and", "is", "for" etc.
   - Lowercase all terms
   - Filter against expanded blocklist (1,391 common English words):
     Verbs: "have", "make", "take", "know", "think"...
     Adjectives: "good", "new", "first", "last", "long"...
     Prepositions: "about", "after", "before", "between"...
   - Count frequency with Counter

3. FREQUENCY BOOSTING (domain signal detection):
   - ALL-CAPS words (e.g., "HYDRAULIC"): +2 frequency bonus
     Why? Technical manuals often capitalize key terms
   - Hyphenated terms (e.g., "cross-platform"): +1 bonus
     Why? Compound technical terms are usually hyphenated
   - Mid-sentence capitalized words (e.g., "Bearing" not at sentence start): +1 bonus
     Why? Proper nouns and technical terms keep capitalization mid-sentence

4. MULTI-WORD TERM EXTRACTION:
   - Pattern: r'\b([A-Z][a-z]+(?:\s+[A-Z][a-z]+){1,2})\b'
   - Matches capitalized bigrams/trigrams:
     "Hydraulic Pump", "Control Valve Assembly", "Landing Gear"
   - Count occurrences separately

5. FREQUENCY FILTER:
   - Only keep terms with frequency >= 5
     Why 5? Filters one-off mentions. Terms appearing 5+ times
     are likely actually significant to the document.

6. SORT by frequency descending

7. CONTEXT EXTRACTION (for top 200 terms):
   - Per page: Split into sentences: re.split(r'(?<=[.!?])\s+', text)
   - Skip sentences < 10 chars
   - For each of top 200 terms:
     - Case-insensitive search in each sentence
     - Collect up to 5 context sentences per term
     - Cap sentence length at 500 chars

   Why top 200? Memory/performance balance. Most documents have
   50-300 significant terms; 200 covers the important ones
   without processing thousands of low-frequency words.

8. Store:
   - Terminology records: {term, frequency, first_letter}
   - TermContext records: {term, sentence, page_number}
```

**Output**: `Terminology` + `TermContext` records in DB

---

### LLM Definitions (Lazy, On-Demand)

**Library**: llama-cpp-python (CPU inference)
**Model**: Phi-3 Mini GGUF (Q4_K_M quantization, ~2.3GB)
**File**: `llm_service.py`

Definitions are NOT generated during the pipeline — they're generated on-demand when a user first clicks a term or hotspot:

```
Architecture:
- Lazy singleton: Model loaded on first API call (not at startup)
- Thread-safe: threading.Lock() for model init
- Serialized generation: _gen_lock ensures one-at-a-time
  (llama.cpp is NOT thread-safe for concurrent generation)
- Broken model detection: If GGML_ASSERT or llama_decode errors occur,
  LLM is disabled for the session (avoids repeated crashes)

Generation Flow:
1. User clicks term "Hydraulic Pump" (or hotspot label)
2. API: GET /documents/{id}/term-info/Hydraulic Pump
3. Check cache: SELECT from term_definitions WHERE doc_id=X AND term='Hydraulic Pump'
4. If cached → return immediately (no LLM call)
5. If not cached:

   a. UNIVERSAL DEFINITION:
      Prompt: "Define the technical term 'Hydraulic Pump' in one concise sentence.
               Focus on its general engineering/technical meaning."
      Format: <|user|>\n{prompt}\n<|end|>\n<|assistant|>  (Phi-3 chat format)
      Parameters: temperature=0.3, max_tokens=200

   b. CONTEXTUAL DEFINITION (if usage sentences available):
      Prompt: "Given these usages of 'Hydraulic Pump':
               1. 'The hydraulic pump supplies 3000 PSI to System A'
               2. 'Inspect hydraulic pump mounting bolts at 500-hour intervals'
               Define this term in the specific context of this document."

   c. NUMERIC LABEL RESOLUTION:
      If the term is a digit (e.g., hotspot label "8"):
      → Query: SELECT component_name FROM hotspots WHERE label_number='8'
      → If found (e.g., "Hydraulic Pump") → use as query_term instead
      → Generate definition for "Hydraulic Pump" not "8"

   d. Cache result in TermDefinition table (unique constraint on doc_id+term)
   e. Handle race conditions: UNIQUE violation → re-fetch + update

Parameters:
  - Max prompt length: 1500 chars (context safety for 4096-token model)
  - Temperature: 0.3 (near-deterministic, factual output)
  - Max output tokens: 200 (concise definitions)
  - Stop tokens: ["<|end|>", "<|user|>"]
```

---

### Interactive Hotspot Rendering on PDF Pages (Frontend)

**File**: `PdfManualViewer.tsx`
**Libraries**: pdf.js, React

This is how the red numbered circles appear on the PDF page:

```
Data Flow:
1. PdfManualViewer loads → fetches documentsApi.getDiagrams(docId)
2. API returns all diagrams with:
   - page_x, page_y, page_w, page_h (diagram bbox as % of page)
   - hotspots[]: each with label_number, x, y (as % within diagram)
3. Build Map<pageNumber, DiagramInfo[]> for quick lookup

Rendering (per page):
1. PdfPage component renders PDF page on <canvas> via pdf.js
2. Canvas is wrapped in a <div style="position: relative">
3. For each diagram on this page that has bbox data:
   For each hotspot in that diagram:

   POSITION CALCULATION:
     absX = diagram.page_x + (hotspot.x / 100) * diagram.page_w
     absY = diagram.page_y + (hotspot.y / 100) * diagram.page_h

     Example:
       Diagram is at page_x=54%, page_y=27%, page_w=36%, page_h=30%
       Hotspot is at x=77%, y=34% within the diagram
       absX = 54 + (77/100)*36 = 54 + 27.7 = 81.7%  (from left of page)
       absY = 27 + (34/100)*30 = 27 + 10.2 = 37.2%  (from top of page)

   RENDER:
     <div style="position: absolute; left: 81.7%; top: 37.2%;
                 width: 22px; height: 22px; border-radius: 50%;
                 background: rgba(220,38,38,0.9); color: white;
                 transform: translate(-50%, -50%)">
       8   ← label number
     </div>

   INTERACTION:
     - Hover: scale(1.25) + red glow shadow
     - Click: Opens HotspotInfoPanel (floating card, right side)
     - HotspotInfoPanel fetches getTermInfo(docId, componentName)
     - Shows: label number, component name, figure info,
              universal definition, contextual definition,
              usage contexts with page numbers

Camera Badge:
  - Top-right corner of each page shows diagram count
  - Click → navigates to dedicated Intelligence page
```

---

## Database Schema — PDF Intelligence Tables (6 tables)

```
┌──────────────────────────────────────────────────────────────────┐
│                       documents                                  │
│  id (UUID PK) │ file_path │ status │ xml_tree (JSONB for TOC)   │
└────────┬─────────────────────────────────────────────────────────┘
         │ CASCADE
         ├─────────────────────────────────────────────────┐
         │                                                 │
         ▼                                                 ▼
┌─────────────────────┐                    ┌─────────────────────────┐
│    pdf_sections      │                    │      terminology         │
│  section_title       │                    │  term: VARCHAR(300)      │
│  content_text: TEXT  │                    │  frequency: INT          │
│  page_start: INT     │                    │  first_letter: CHAR(1)   │
│  page_end: INT       │                    │  UNIQUE(doc_id, term)    │
└──────────┬──────────┘                    └─────────────────────────┘
           │ SET NULL                                │
           ▼                                         ▼
┌──────────────────────────┐           ┌──────────────────────────┐
│       diagrams            │           │     term_contexts         │
│  page_number: INT         │           │  term: VARCHAR(300)       │
│  image_path: VARCHAR      │           │  sentence: TEXT           │
│  width, height: INT       │           │  page_number: INT         │
│  figure_name: VARCHAR     │           └──────────────────────────┘
│  figure_title: VARCHAR    │                        │
│  has_callouts: BOOLEAN    │                        ▼
│  image_hash: VARCHAR(64)  │           ┌──────────────────────────┐
│  page_x: FLOAT (%)       │           │    term_definitions       │
│  page_y: FLOAT (%)       │           │  term: VARCHAR(300)       │
│  page_w: FLOAT (%)       │           │  universal_definition     │
│  page_h: FLOAT (%)       │           │  contextual_definition    │
└──────────┬───────────────┘           │  UNIQUE(doc_id, term)    │
           │ CASCADE                    └──────────────────────────┘
           ▼
┌──────────────────────────┐
│       hotspots            │
│  label_number: VARCHAR    │
│  component_name: VARCHAR  │
│  x: FLOAT (%)            │  ← position within diagram image
│  y: FLOAT (%)            │
│  width: FLOAT (%)        │
│  height: FLOAT (%)       │
└──────────────────────────┘
```

### Alembic Migrations (6)

| Migration | Description |
|-----------|-------------|
| `001_initial.py` | All core tables (users, roles, fleets, manuals, documents, etc.) |
| `002_xml_viewer.py` | `xml_tree` JSONB column, `has_images` flag on documents |
| `003_fleet_hierarchy.py` | Fleet parent_id, image_path, platform_type |
| `004_pdf_intelligence.py` | PdfSection, Diagram, Hotspot, TermContext, TermDefinition, Terminology tables |
| `005_diagram_enhancements.py` | figure_name, figure_title, has_callouts, image_hash columns |
| `006_diagram_page_bbox.py` | page_x, page_y, page_w, page_h columns on diagrams |

---

## Key Thresholds & Parameters Reference

| Parameter | Value | Location | Rationale |
|-----------|-------|----------|-----------|
| Diagram min size | 250 x 250 px | `_extract_diagrams_with_progress()` | Filters icons, logos, bullets |
| Caption proximity | 150 pt below image | `extract_figure_captions()` | ~2 inches, catches most caption styles |
| Callout preview downscale | 600px width | `has_numeric_callouts()` | Fast screening, 3-5x speedup |
| Hotspot OCR downscale | 1200px width | `detect_hotspots()` | Accurate positions, 2-3x speedup |
| OCR confidence threshold | 0.3 | Both OCR functions | EasyOCR's 0-1 scale; 0.3 filters noise |
| Callout decision | ≥2 distinct digits | `has_numeric_callouts()` | Single digit could be page/part number |
| Label range | 1-99 | All label extraction | Standard callout numbering range |
| Thread pool workers | 4 | Parallel OCR functions | CPU-bound tasks, 4 cores typical |
| Callout check timeout | 60s | `run_callout_checks_parallel()` | Handles first-run model loading |
| Hotspot detection timeout | 30s | `run_hotspot_detection_parallel()` | Model already loaded from Step 4 |
| Terminology min frequency | 5 | `extract_terminology()` | Filters one-off mentions |
| Term min length | 4 chars | `extract_terminology()` | Filters "the", "and", etc. |
| Top terms for context | 200 | `_run_pipeline()` Step 7 | Memory/performance balance |
| Max contexts per term | 5 | `extract_term_contexts()` | Representative sampling |
| Context sentence max | 500 chars | `extract_term_contexts()` | Prevents storing paragraphs |
| LLM prompt max | 1500 chars | `llm_service._generate()` | Context safety for 4096 token model |
| LLM temperature | 0.3 | `llm_service._generate()` | Near-deterministic, factual |
| LLM max output | 200 tokens | `llm_service._generate()` | Concise definitions |
| LLM context length | 4096 | Config | Phi-3 Mini native capacity |
| Frequency boost: ALL-CAPS | +2 | `extract_terminology()` | Technical manuals capitalize key terms |
| Frequency boost: hyphenated | +1 | `extract_terminology()` | Compound technical terms |
| Frequency boost: mid-cap | +1 | `extract_terminology()` | Proper nouns / technical terms |

---

## End-to-End System Flow (Updated)

```
                        IETM Platform — Complete Flow (2026)
 ============================================================================

 [1] USER UPLOADS DOCUMENT (PDF / XML / ZIP)
     POST /api/documents/upload  (with JWT Bearer token)
     File saved to: storage/uploads/<manual_id>/<uuid>.<ext>
                          │
              ┌───────────┴───────────┐
              │                       │
         PDF document            XML/ZIP document
              │                       │
              ▼                       ▼
 [2a] PDF PROCESSING              [2b] XML PROCESSING
     ┌─────────────────┐          ┌─────────────────────┐
     │ Extract TOC via │          │ Parse with           │
     │ PyMuPDF outline │          │ defusedxml           │
     │ Store in JSONB  │          │ (or S1000D parser    │
     └────────┬────────┘          │  for BREX/CSDB/ICN)  │
              │                    │ Build tree + xrefs   │
              ▼                    │ Store in xml_tree    │
 [3] PDF INTELLIGENCE PIPELINE    └─────────────────────┘
     (Background asyncio task)
     ┌─────────────────────────────────────────────┐
     │ Step 0: PyMuPDF get_text("blocks")          │
     │         → Section extraction                │
     │                                             │
     │ Step 1: PyMuPDF get_images + extract_image  │
     │         → 250px filter → SHA256 dedup       │
     │         → get_image_rects → page bbox %     │
     │                                             │
     │ Step 2: PyMuPDF get_text("dict")            │
     │         → Scan 150pt below each image       │
     │         → Regex match "Fig. X-Y"            │
     │                                             │
     │ Step 3: pdfplumber extract_tables()         │
     │         → Match fig refs to titles          │
     │                                             │
     │ Step 4: PIL resize 600px + EasyOCR          │
     │         → Parallel callout screening (4T)   │
     │         → ≥2 digits = has_callouts          │
     │                                             │
     │ Step 5: PIL resize 1200px + EasyOCR         │
     │         → Parallel hotspot detection (4T)   │
     │         → Label positions as % coords       │
     │                                             │
     │ Step 6: pdfplumber tables + regex           │
     │         + circled ①② number support         │
     │         → Map label → component name        │
     │                                             │
     │ Step 7: Regex frequency analysis + NLP      │
     │         → Terminology + context sentences   │
     │                                             │
     │ (LLM definitions generated on-demand later) │
     └─────────────────────────────────────────────┘
                          │
                          ▼
 [4] DOCUMENT READY FOR VIEWING
     ┌─────────────────────────────────────────────┐
     │ PDF Viewer: pdf.js canvas + hotspot overlay │
     │   → Red circles at label positions          │
     │   → Click → floating panel with definition  │
     │   → Camera badge → Intelligence page        │
     │                                             │
     │ XML Viewer: Sidebar tree + rich renderer    │
     │   → Paragraphs, warnings, procedures        │
     │   → Figures with zoom, tables, xref pills   │
     │   → Related sections panel                  │
     │                                             │
     │ Intelligence Page:                          │
     │   → Diagrams tab: grid of extracted images  │
     │   → Terminology tab: A-Z with definitions   │
     │   → Click diagram → hotspot detail modal    │
     └─────────────────────────────────────────────┘
```

---

## Phase 1: Content Intelligence Pipeline (COMPLETE)

### Pipeline Architecture

```
Upload PDF/XML
        │
        ▼
  +-----------+     +------------------+     +-------------------+
  | EXTRACT   | --> | DETECT HEADINGS  | --> | SEGMENT INTO      |
  | (text +   |     | (font size,      |     | SECTIONS          |
  |  metadata)|     |  bold, ATA codes)|     | (heading + body)  |
  +-----------+     +------------------+     +-------------------+
                                                      │
        +---------------------------------------------+
        │
        ▼
  +------------------+     +---------------------+     +------------------+
  | DETECT ROOT      | --> | CLASSIFY SECTIONS   | --> | DETECT REFS      |
  | SYSTEM           |     | (Hybrid: ATA code + |     | (procedures,     |
  | (aircraft/naval/ |     |  rules + keywords + |     |  figures, tables, |
  |  ground vehicle) |     |  AI semantic)       |     |  cross-refs)     |
  +------------------+     +---------------------+     +------------------+
                                    │
                                    ▼
                           +------------------+
                           | GENERATE TREE    |
                           | (hierarchical    |
                           |  document tree)  |
                           +------------------+
```

### Phase 1 Components

| Component | File | Lines | Purpose |
|-----------|------|-------|---------|
| Taxonomy Engine | `ietm_taxonomy.py` | 1,865 | 79 categories, 1,391 keywords, ATA mappings |
| Data Models | `schemas/intelligence.py` | 302 | All pipeline data structures |
| PDF Extractor | `extractors/pdf_extractor.py` | 215 | PyMuPDF text + font metadata |
| XML Extractor | `extractors/xml_extractor.py` | 241 | defusedxml, S1000D support |
| Heading Detector | `detectors/heading_detector.py` | 353 | 5-signal heading detection |
| Reference Detector | `detectors/reference_detector.py` | 245 | Procedures, figures, cross-refs |
| Embedding Service | `classifiers/embedding_service.py` | 315 | AI model (all-MiniLM-L6-v2, 384-dim) |
| Hybrid Classifier | `classifiers/hybrid_classifier.py` | 521 | 3-method classification engine |
| Root Detector | `classifiers/root_detector.py` | 124 | Aircraft/naval/ground detection |
| Tree Generator | `services/tree_generator.py` | 170 | Hierarchical tree builder |
| Pipeline Orchestrator | `services/content_intelligence.py` | 261 | Main entry point |
| Tests | `tests/test_pipeline.py` | 685 | 26/26 tests passing |

---

## Phase 2: Backend Platform (COMPLETE)

### What Was Built

46 files implementing the complete web platform:

| Layer | Components | Details |
|-------|-----------|---------|
| **Database** | PostgreSQL + pgvector + Alembic | 19 tables, UUID PKs, JSONB, full-text search, vector embeddings |
| **Authentication** | JWT + bcrypt + Sessions | 15min access tokens, 7-day refresh (httpOnly cookie), session tracking |
| **Authorization** | RBAC (4 roles, 17 permissions) | super_admin, fleet_admin, editor, viewer + fleet-level access control |
| **API Endpoints** | 30+ REST endpoints | Auth, Users, Fleets, Manuals, Documents, Intelligence, SSE progress |
| **Security** | Middleware stack | Security headers (CSP, HSTS, X-Frame), rate limiting (30r/s), audit logging |

### Database Schema (19 Tables)

```
┌──────────────┐     ┌──────────────┐     ┌──────────────────┐
│    users      │────>│  user_roles   │<────│     roles        │
│  (UUID PK)    │     │  (M2M join)   │     │  (4 default)     │
└──────┬───────┘     └──────────────┘     └────────┬─────────┘
       │                                           │
       │                                  ┌────────┴─────────┐
       │                                  │ role_permissions  │
       │                                  │   (M2M join)     │
       │                                  └────────┬─────────┘
       │                                           │
       │                                  ┌────────┴─────────┐
       │                                  │   permissions     │
       │                                  │  (17 default)     │
       │                                  └──────────────────┘
       │
       ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────────┐
│ fleet_access  │────>│   fleets      │<────│    manuals       │
│ (user+fleet)  │     │  (hierarchy)  │     │  (per fleet)     │
└──────────────┘     └──────────────┘     └────────┬─────────┘
                                                    │
                                           ┌────────┴─────────┐
                                           │    documents      │
                                           │  (file_path,      │
                                           │   xml_tree JSONB,  │
                                           │   tsvector + GIN)  │
                                           └────────┬─────────┘
                                                    │
                               ┌────────────────────┼────────────────────┐
                               │                    │                    │
                               ▼                    ▼                    ▼
                    ┌────────────────┐    ┌──────────────┐    ┌──────────────────┐
                    │  pdf_sections   │    │   diagrams    │    │   terminology     │
                    │  (text content) │    │  (images +    │    │  (term frequency) │
                    └────────────────┘    │   bbox + OCR) │    └──────────────────┘
                                          └──────┬───────┘
                                                 │
                                          ┌──────┴───────┐
                                          │   hotspots    │
                                          │  (label pos + │
                                          │   component)  │
                                          └──────────────┘
```

### API Endpoints (Complete)

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/api/auth/login` | Public | Login (returns JWT) |
| `POST` | `/api/auth/logout` | Bearer | Logout (invalidates session) |
| `POST` | `/api/auth/refresh` | Cookie | Refresh access token |
| `GET` | `/api/auth/me` | Bearer | Current user profile |
| `GET` | `/api/users` | Admin | List all users |
| `POST` | `/api/users` | Admin | Create user |
| `PUT` | `/api/users/{id}` | Admin | Update user |
| `DELETE` | `/api/users/{id}` | Admin | Deactivate user |
| `GET` | `/api/fleets` | Bearer | List fleets |
| `POST` | `/api/fleets` | Admin | Create fleet |
| `PUT` | `/api/fleets/{id}/image` | Admin | Upload fleet image |
| `GET` | `/api/manuals` | Bearer | List manuals |
| `POST` | `/api/manuals` | Editor+ | Create manual |
| `POST` | `/api/documents/upload` | Editor+ | Upload PDF/XML |
| `POST` | `/api/documents/upload-zip` | Editor+ | Upload XML manual as ZIP |
| `GET` | `/api/documents/{id}` | Bearer | Get document metadata |
| `GET` | `/api/documents/{id}/file` | Bearer | Serve original file |
| `GET` | `/api/documents/{id}/xml-tree` | Bearer | Get parsed XML/PDF TOC tree |
| `GET` | `/api/documents/{id}/related/{node}` | Bearer | Cross-reference graph |
| `GET` | `/api/documents/{id}/asset/{path}` | Bearer | Serve extracted assets |
| `GET` | `/api/documents/{id}/sections` | Bearer | PDF extracted sections |
| `GET` | `/api/documents/{id}/diagrams` | Bearer | Diagrams + hotspots + bbox |
| `GET` | `/api/documents/{id}/diagram-image/{id}` | Public | Serve diagram image file |
| `GET` | `/api/documents/{id}/terminology` | Bearer | Terminology list (A-Z filter) |
| `GET` | `/api/documents/{id}/term-info/{term}` | Bearer | LLM definitions + contexts |
| `POST` | `/api/documents/{id}/reprocess-intelligence` | Admin | Re-run pipeline |
| `GET` | `/api/documents/progress/{id}` | Public | SSE pipeline progress stream |
| `GET` | `/api/documents/intelligence/diagrams` | Bearer | Cross-document diagram search |
| `GET` | `/api/documents/intelligence/terminology` | Bearer | Cross-document terminology |
| `GET` | `/api/explorer/tree` | Bearer | Cross-fleet document explorer |

### Security Middleware Stack

```
Request → [Rate Limiter (30r/s)] → [Security Headers] → [Audit Logger] → Route Handler
                                         │
                                         ├─ X-Frame-Options: DENY
                                         ├─ X-Content-Type-Options: nosniff
                                         ├─ X-XSS-Protection: 1; mode=block
                                         ├─ Strict-Transport-Security: max-age=31536000
                                         ├─ Content-Security-Policy: default-src 'self'
                                         └─ Referrer-Policy: strict-origin-when-cross-origin
```

---

## Phase 3: Frontend (COMPLETE)

### What Was Built

Full React SPA with Vite + TypeScript:

| Feature | Details |
|---------|---------|
| **Auth** | Login page, JWT token management, route guards, auto-refresh |
| **Fleet Management** | Fleet list with images, create/delete, image upload |
| **Manual Management** | Manual list per fleet, create/delete, document association |
| **Document Upload** | PDF, XML, and ZIP upload with live progress streaming |
| **PDF Viewer** | pdf.js canvas rendering, zoom, TOC sidebar, interactive hotspot overlay |
| **XML Viewer** | Sidebar tree navigation, rich content rendering, image zoom |
| **Intelligence Page** | Diagrams grid + terminology browser with LLM definitions |
| **User Management** | User list, create, role assignment, deactivate |
| **Explorer** | Cross-fleet/manual document explorer with restructure |

### Frontend File Structure

```
frontend/src/
├── main.tsx, App.tsx
├── api/
│   ├── client.ts                     # Axios + JWT interceptors
│   ├── types.ts                      # All TypeScript interfaces (30+)
│   └── endpoints.ts                  # API function wrappers
├── components/ui/
│   └── Spinner.tsx                   # Loading indicators
├── hooks/
│   └── useApiData.ts                 # Generic async data fetcher
└── pages/
    ├── Login.tsx, Dashboard.tsx
    ├── Fleets.tsx, Manuals.tsx, Documents.tsx
    ├── Users.tsx, Explorer.tsx
    ├── intelligence/
    │   └── IntelligencePage.tsx       # Diagrams grid + terminology browser
    └── viewer/
        ├── PdfManualViewer.tsx        # PDF viewer + hotspot overlays + info panel
        ├── XmlManualViewer.tsx        # XML viewer (sidebar + content)
        ├── XmlContentRenderer.tsx     # Rich content renderer
        └── RelatedSectionsPanel.tsx   # Cross-reference navigation
```

---

## Technology Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| API Framework | FastAPI | Async, auto-docs, type-safe |
| Database | PostgreSQL 16 + pgvector | JSONB, full-text search, vector similarity |
| ORM | SQLAlchemy 2.0 (async) | Modern async pattern with asyncpg driver |
| Migrations | Alembic | Async migration support, 6 revisions |
| Auth | JWT (python-jose) + bcrypt | Industry standard, stateless tokens |
| AI Model | all-MiniLM-L6-v2 | 384-dim embeddings, CPU, fully offline |
| LLM | Phi-3 Mini GGUF (Q4_K_M) | Offline definitions, CPU-only, 2.3GB |
| PDF Parsing | PyMuPDF (fitz) | Fast, font metadata, image extraction, bbox |
| PDF Tables | pdfplumber | Structured table extraction from PDFs |
| OCR | EasyOCR | CPU-only, digit detection, bbox output |
| Image Processing | PIL/Pillow + numpy | Resizing for OCR, array conversion |
| XML Parsing | defusedxml | Prevents XXE attacks (critical for defense) |
| NLP | NLTK | Stopwords for terminology filtering |
| Frontend | React 18 + Vite + TypeScript | Fast builds, type safety |
| PDF Viewing | pdf.js | Canvas rendering, page navigation |
| Icons | Lucide React | Camera, zoom, folder icons |
| HTTP Client | Axios | JWT interceptors, file upload |

---

## Project Directory Structure

```
IETM/
├── PLAN.md                                    # Implementation plan
├── PROJECT_STATUS.md                          # This file
├── ietm_taxonomy.py                           # Taxonomy engine (1,865 lines)
├── .env                                       # Environment configuration
├── storage/
│   ├── uploads/                               # Uploaded documents
│   │   └── diagrams/{doc_id}/                 # Extracted diagram images
│   ├── manuals/{doc_id}/                      # Extracted XML manuals + assets
│   └── fleet_images/                          # Fleet/platform images
│
├── backend/
│   ├── alembic/versions/
│   │   ├── 001_initial.py                     # Core tables
│   │   ├── 002_xml_viewer.py                  # xml_tree JSONB
│   │   ├── 003_fleet_hierarchy.py             # Fleet hierarchy
│   │   ├── 004_pdf_intelligence.py            # Intelligence tables
│   │   ├── 005_diagram_enhancements.py        # figure_name, has_callouts, image_hash
│   │   └── 006_diagram_page_bbox.py           # page_x/y/w/h for hotspot overlay
│   │
│   ├── ml_models/
│   │   ├── all-MiniLM-L6-v2/                  # Embedding model (384-dim)
│   │   └── phi3-mini-Q4_K_M.gguf             # LLM for definitions (2.3GB)
│   │
│   └── app/
│       ├── models/
│       │   ├── pdf_intelligence.py            # PdfSection, Diagram, Hotspot,
│       │   │                                  # TermContext, TermDefinition, Terminology
│       │   ├── user.py, fleet.py, manual.py, document.py, audit.py, taxonomy.py
│       │   └── ...
│       │
│       ├── services/
│       │   ├── pdf_intelligence_service.py    # 8-step pipeline orchestrator
│       │   ├── pdf_content_extractor.py       # PyMuPDF text/section extraction
│       │   ├── diagram_extractor.py           # Image extraction + EasyOCR hotspots
│       │   ├── label_table_extractor.py       # pdfplumber label→component mapping
│       │   ├── terminology_extractor.py       # NLP frequency analysis + contexts
│       │   ├── llm_service.py                 # Phi-3 GGUF offline definitions
│       │   ├── progress_store.py              # SSE progress tracking
│       │   ├── xml_manual_parser.py           # XML → JSON tree + xref graph
│       │   ├── s1000d_parser.py               # S1000D dataset parser
│       │   ├── document_service.py            # Upload + pipeline trigger
│       │   └── ...
│       │
│       ├── schemas/
│       │   ├── pdf_intelligence.py            # DiagramResponse, HotspotResponse, etc.
│       │   └── ...
│       │
│       └── api/
│           ├── documents.py                   # 30+ endpoints including intelligence
│           └── ...
│
└── frontend/src/
    ├── api/types.ts                           # 30+ TypeScript interfaces
    ├── api/endpoints.ts                       # API wrappers
    └── pages/
        ├── intelligence/IntelligencePage.tsx   # Diagram grid + terminology browser
        └── viewer/PdfManualViewer.tsx          # PDF viewer + interactive hotspots
```

---

## How to Run

### Prerequisites
- Python 3.11+
- PostgreSQL 16 with pgvector extension
- Node.js 18+ (for frontend)

### Quick Start

```bash
# 1. Backend
cd backend
pip install -r requirements.txt
alembic upgrade head
uvicorn backend.app.main:app --host 0.0.0.0 --port 8000 --reload

# 2. Frontend
cd frontend
npm install
npm run dev

# 3. First login
# Username: admin, Password: (DEFAULT_ADMIN_PASSWORD from .env)
```

### To Test Hotspots
1. Login as admin
2. Create a fleet and manual
3. Upload a technical PDF with numbered diagram callouts
4. Wait for pipeline to complete (watch progress on Documents page)
5. Open the PDF in the viewer
6. Scroll to a page with diagrams — look for red numbered circles
7. Click a circle to see the component name and definition

---

## What's Next

- Full-text + semantic search with result highlighting
- Content authoring / inline editor
- PDF export with annotations
- Audit dashboard with usage analytics
