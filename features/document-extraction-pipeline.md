# Document Extraction Pipeline (OCR → Section Classification → Streamed Structured Extraction)

## Feature Specification for Porting

This document describes a **production document-extraction pipeline**. It takes any PDF or image, from a 4-page form to a 100-page multi-document packet, and returns structured JSON per document section. Progress and partial results stream back as they are produced. Results are cached by content hash, and every run ends with an explicit, honest status. Two front doors share it: a portal upload (the user watches live progress) and an email agent (a server-side worker consumes the same stream).

The reference implementation reads insurance submission packets: ACORD applications, supplemental questionnaires, carrier loss runs and broker summaries. It is split between a Node/Express API (intake, records, storage) and a Python/FastAPI **extraction worker** (OCR, classification, model calls), uses **Mistral OCR** for page reading and a Mistral chat model for JSON extraction, and stores everything in MongoDB. The pattern ports to any "packet of mixed documents → typed records" problem: invoices + receipts, KYC bundles, medical records, legal discovery, RFP responses.

> **Companion spec:** [`email-intake-agent.md`](./email-intake-agent.md) feeds this pipeline from a shared mailbox and replies with what is still missing.

---

## 1. Overview

```
  Portal upload ─┐                       ┌──────────── EXTRACTION WORKER (FastAPI) ─────────────┐
                 │  intake (API)         │                                                       │
  Email agent ───┼─▶ hash · dedupe ──────┼─▶ POST /extract/auto-stream   (one file per call)    │
                 │   upload rows         │    ├─ image → PDF                                     │
                 │   multipart form      │    ├─ cache? (hash + tenant + versions) ──▶ replay    │
                 │                       │    ├─ OCR (cached by hash) ── page-delimited text     │
                 │                       │    ├─ classify pages → sections with page ranges     │
                 │                       │    ├─ extract each section in parallel (only its pages)│
                 │                       │    │    token stream → incremental JSON → key chunks   │
                 │                       │    ├─ validate → status                               │
                 │                       │    └─ persist (awaited) ── then complete              │
                 │                       └───────────────────────┬───────────────────────────────┘
                 │                                               │ NDJSON events
                 ▼                                               ▼
       ┌─────────────────────┐   finalize   ┌──────────────────────────────────────────┐
       │ API stream consumer │◀─────────────│ progress · classification · sub_chunk ·  │
       │ (browser or worker) │              │ row · type_complete · warning · complete │
       └─────────┬───────────┘              └──────────────────────────────────────────┘
                 ▼
     merge into record → auto-map to the data model → "ready" / "needs info"
```

### Design principles

| # | Principle | Why |
|---|-----------|-----|
| 1 | **Content hash is identity.** SHA-256 of the file bytes keys the upload dedupe, the OCR cache and the final-result cache. | The same PDF arrives by upload, by email and as a resend. It is read once. |
| 2 | **Cache in layers, invalidate by version.** OCR text is reused whatever the prompt version. The final JSON is reused only when the prompt, schema and extraction versions all match. | Changing a prompt re-runs only the cheap part. OCR is the slow, expensive step. |
| 3 | **Classify pages, then send each extractor only its pages.** | A loss-run prompt never sees the application form. Token budgets scale with the section, not the packet. |
| 4 | **Stream everything.** NDJSON events per stage, per section and per completed JSON key. | The UI shows progress on a 3-minute job, and a worker can tell "still going" from "hung". |
| 5 | **The status is honest.** `completed` / `partial_success` / `partial_failed` / `failed` / `insufficient`. A non-domain document is kept but flagged unusable. | Downstream logic (gap analysis, quoting) must not treat "the model returned something" as "the document was read". |
| 6 | **Save first, then announce.** The final result is written *before* the `complete` event is emitted. | Consumers react to `complete` by reading the stored record. If the write is still in flight they get a 404 (this happened when the save was a fire-and-forget task). |
| 7 | **One file per worker call.** | Per-file status, per-file retries, per-file cache. A packet of 5 files is 5 independent jobs. |

---

## 2. Architecture

| Layer | Responsibility | Reference |
|-------|----------------|-----------|
| **Intake (API)** | Hash files, check for duplicates, create upload and session rows, build the multipart request, consume the stream, merge results into the record | `intakeService.processIntakePacket()` (Node) |
| **Extraction worker** | Normalize the file, cache lookup, OCR, page classification, per-section extraction, validation, persistence | `POST /extract/auto-stream` (FastAPI) |
| **Prompts** | One prompt file per section type plus a classification prompt, versioned | `prompts/<type>_extraction.yaml` |
| **Model gateway** | Global concurrency limit, retries, usage/cost logging for every model call | `mistral_extraction_service` |
| **Post-processing (API)** | Merge sections across files into one record view, auto-map to the record's data model, compute readiness | `autoMapAfterExtraction()` |

The API and worker share the same MongoDB collection for extraction results (`document_extractions`). The worker writes it, and the API reads it for duplicate checks and record views.

---

## 3. Intake (API side)

```
processIntakePacket({ files, body: { existingApplicationId? }, tenantId, userId, source })
  1. for each file: hash = sha256(bytes)
  2. duplicate check (tenant-scoped): an upload or extraction with this hash already exists?
       → isCacheHit = { applicationId, hasExtraction }
  3. resolve the target record: existingApplicationId, else the record owning the duplicate, else create a placeholder
     ("Extracting from <file>…", renamed once the documents reveal the real name)
  4. create a SubmissionSession (batch) + one DocumentUpload row per file
     { applicationId, fileName, s3Url, documentHash, extractionStatus: 'processing'|'completed', source }
  5. build the multipart form for the worker:
     files, tenantId, userId, applicationId, submissionSessionId, documentsManifest (JSON),
     documentUploadId(s), force_reextract?, force_reprocess?
  → { applicationId, submissionSessionId, documentsManifest, documentHashes, isCacheHit, pythonForm }
```

- **Byte-identical duplicate that is already extracted:** don't call the worker. Mark the upload `completed`, close the session as `completed_success`, and `$addToSet` the new upload id onto the existing extraction's `documentUploadIds`. One extraction can then serve many uploads.
- **Size cap by bytes, never by page count.** A 100-page packet is normal. A 300 MB scan is not. (The email path caps at 25 MB per attachment.)
- `source` (`broker_portal`, `email_ingestion`, …) is recorded on every row so you can tell which front door a document came through.

---

## 4. The Extraction Worker

### 4.1 Normalize

Write the upload to a temp file. If it is an image (`.png`, `.jpg`, `.jpeg`, `.webp`), convert it to a one-page PDF (PyMuPDF `convert_to_pdf()`) so everything downstream deals with PDFs only. Temp files are always removed in a `finally`.

### 4.2 Layered cache

```
hash = sha256(content)
force_reextract → delete every cache layer for (hash, tenant) first — a true fresh run
force_reprocess → skip the result cache (layer 2) and OCR cache, re-run the pipeline

Layer 2 — final result   key: (hash, tenant)   valid only if promptVersion, schemaVersion,
                                                extractionVersion all equal CURRENT_*
          hit → replay as a stream: classification → extraction_chunk per type → type_complete → complete
Layer 1 — OCR text       key: (hash, tenant)   version-independent
          hit → rebuild pages from the stored page-delimited markdown, skip OCR
```

Bump a version constant (`CURRENT_PROMPT_VERSION`, …) whenever you change a prompt or the output schema. Every cached result goes stale automatically, while the OCR layer stays warm.

Replaying the cached result **as the same event stream** means a consumer never needs a separate "cached" code path.

### 4.3 OCR: two engines, selected by `PDF_ENGINE`

| Engine | How | When |
|--------|-----|------|
| `mistral_ocr` | Send the PDF to an OCR model that returns **Markdown per page** (tables, checkboxes, form fields, radio buttons). Store it as one document with `--- PAGE N START --- … --- PAGE N END ---` delimiters ("frozen markdown"). | Default. Best on forms and scans. A failure ends the stream with an `error` event (there is no silent fallback). |
| `native` | Read the embedded text layer (PyPDF2). **Render a page to an image** for the vision model when its text can't be trusted: (a) under `OCR_MIN_TEXT_CHARS` (scanned), (b) alphanumeric ratio under `OCR_MIN_ALNUM_RATIO` (mojibake text layer), (c) a dense numeric grid (`OCR_IMAGE_FOR_TABLES`), because linearized table text destroys row/column alignment and the model pairs values with the wrong rows, or (d) every page (`OCR_ALWAYS_IMAGE`). Render at `OCR_RENDER_DPI`. | Cheap path for digital PDFs |

Rendering runs in a thread executor so the async event loop never blocks on CPU-bound PDF work.

### 4.4 Section classification: pages → typed page ranges

Every page is assigned to exactly one section type, and a type may cover non-contiguous ranges:

```json
{ "summary": [1], "acord": [2,3,4,5], "supplemental": [6,7], "loss_run": [27,28,29,30],
  "generic": { "bureau_report": [8], "other": [9, 10] } }
```

- **Deterministic first.** In the streaming path a rule-based scorer (`build_section_ranges(page_text_map)`) is authoritative. It scores each page on headings, form numbers and table signatures, and `generic` is the lowest priority. Generic pages carry subtypes, which are flattened into a single `generic` section for extraction.
- **LLM page classifier** (used in the batch path): a heading-first prompt ("classify by the main heading / header / footer first, content second; every page in exactly one section"), 2 attempts. If it still returns nothing usable, it **falls back to running every section type over the full document** rather than failing.
- **Domain-content gate.** If no domain section was detected *and* the text contains no domain markers (tax-ID pattern, domain keywords), the document is not a domain document. Emit a `warning` event, extract everything as `generic` only, attach the **raw OCR text of every page** deterministically (no extra LLM call), and finish as `insufficient`. Nothing that was read is lost, but nothing downstream treats it as a usable document.
- The section ranges are cached alongside the OCR text.

### 4.5 Per-section extraction: parallel, streamed, budgeted

```python
STREAMING_TYPES = {"acord", "supplemental", "loss_run", "summary", "generic"}
sem = asyncio.Semaphore(MAX_CONCURRENT_STREAMS)          # per request (default 3)
queue = asyncio.Queue()                                   # workers → event stream

async def run_section(t, pages):
    async with sem:
        async for chunk in extractor._extract_one_streaming(t, pages, total_pages, text_parts, image_parts):
            await queue.put(chunk)                         # extraction_sub_chunk / row
        await queue.put({"event": "type_complete", "type": t})
```

- **Only the section's own pages** are sent, with the prompt saying *"Extract ALL data from this `<type>` document (pages 2-5 of 46). Only the pages shown below belong to this document."*
- **One prompt per type** (`acord_extraction.yaml`, `loss_run_extraction.yaml`, …), each with its own output schema.
- **Two concurrency limits:** a per-request semaphore (`MAX_CONCURRENT_STREAMS`) and a **process-wide** semaphore on model calls (`MAX_CONCURRENT_MISTRAL_CALLS`). The total stays within the provider's rate limit however many uploads run at once. Keep the two values equal. Starting more workers than the global limit only adds queueing.
- **Budgets scale with the page range:** `max_tokens = clamp(pages × per_type_tokens, 8000, 65535)` (per page: acord 2500, supplemental 2500, loss_run 2000, summary 1500, generic 1200), and `timeout = min(180 + 20 × pages, 900)` seconds.
- **Retry by failure kind:** an empty, truncated (`finish_reason = length`) or unparsable response is retried (up to 2 times) for **only that section**, with its page range.
- **Token streaming + an incremental JSON parser.** The model streams one JSON object. A small parser tracks brace depth and string/escape state and yields each **top-level key** as soon as its value closes:

  ```
  model tokens:  {"applicant": {...}, "locations": [...], "policy": {...}}
  parser emits:  {"key":"applicant","data":{...}}  →  {"key":"locations","data":[...]}  → …
  ```

  The UI fills in field groups as they arrive, and if a stream is cut off partway, the completed keys are still kept. Tabular sections (loss runs) also emit one `row` event per row.
- **Repair before rejecting:** pull the JSON object out of surrounding prose, repair truncated JSON (close open strings, arrays and objects), replace typographic quotes, and strip citation markers the model adds.
- **Domain validators** run after extraction, in both the streaming and batch paths (formats such as email and tax ID, merged locations, mandatory address parts). The validated object is emitted as a final `__validated__` sub-chunk, and consumers replace their accumulated copy with it.

### 4.6 Validation & status

| Status | Meaning | Usable downstream? |
|--------|---------|--------------------|
| `completed` | every section passed quality checks | yes |
| `partial_success` | some sections have validation issues | yes, flagged |
| `partial_failed` | at least one section failed outright | yes for the sections that succeeded |
| `failed` | multiple critical failures | no |
| `insufficient` | read fine, but not a domain document (§4.4) | **no**, but the full text is kept and viewable |

`validate_section_quality(section_results, section_ranges)` decides the first three. Section-level failures upgrade the status to `partial_failed`, and the domain-content gate sets `insufficient`.

### 4.7 Persist, then emit `complete`

The save is **awaited inside the stream generator before `complete` is emitted**. It stores the final per-section JSON (compacted: empty and redundant values stripped), field and row counts, `detectedTypes`, `sectionPageRanges`, the OCR markdown and per-page text, `ocrSource` (`mistral` | `cache` | `native`), the OCR and extraction model ids, processing time, the three version strings, and the upload ids that reference it. A save error is logged and doesn't fail the stream: the consumer still gets the streamed data, and the next run of the same hash simply misses the cache.

> An earlier version ran the save as a fire-and-forget `asyncio.create_task` so it would survive a client disconnect. The UI then fetched the record on `complete` and got 404s. If disconnect-survival matters to you, run the whole extraction as a job (queue + worker) and have the stream report on the job, rather than detaching only the save.

---

## 5. Streaming Protocol (NDJSON)

One JSON object per line, `Content-Type: application/x-ndjson`:

| Event | Payload | When |
|-------|---------|------|
| `processing_started` | `message` | immediately (keeps proxies from timing out) |
| `progress` | `stage` (`init`/`ocr`/`classification`/`extraction`/`saving`), `message`, `percent`, `total_types?`, `type_list?` | at each stage boundary |
| `classification` | detected types + page ranges | after classification (also first on a cache replay) |
| `warning` | `type` (e.g. `not_insurance_document`), `message` | domain-content gate |
| `extraction_sub_chunk` | `type`, `key`, `data` (`key = "__validated__"` carries the full validated object) | each completed top-level key |
| `row` | `type`, row object | tabular sections |
| `extraction_chunk` | `type`, `data` | a whole section at once (cache replay, non-streamed fallback) |
| `type_complete` | `type` | a section finished |
| `error` | `message` | a section or the whole run failed (may still be followed by `complete` = partial success) |
| `complete` | `status: "success"`, `extraction_status` (§4.6), `markdown` (the OCR text, `""` on the native engine), `markdownChars` | after the result is saved |

**Consumer contract:** `complete` means success, even if `error` events came before it (partial success). The stream closing **without** `complete` means failure.

---

## 6. Consuming the Stream (API side)

```js
const res = await axios.post(`${PYTHON_BACKEND_URL}/extract/auto-stream`, form,
  { headers: form.getHeaders(), responseType: 'stream', timeout: 900000 });
let acc = {}, sawComplete = false, err = null;
const rl = readline.createInterface({ input: res.data });
rl.on('line', (line) => {
  const e = safeJson(line); if (!e) return;
  if (e.event === 'extraction_sub_chunk' && e.data)
    e.key === '__validated__' ? (acc = e.data) : (!e.key.startsWith('__') && (acc[e.key] = e.data));
  else if (e.event === 'extraction_chunk') Object.assign(acc, e.data?.data ?? e.data);
  else if (e.event === 'complete') sawComplete = true;
  else if (e.event === 'error') err = e.message;
});
await once(rl, 'close');
await finalize({ success: sawComplete, acc, err });
```

- **Two consumers, one protocol.** The portal proxies the stream to the browser (live progress bar, sections appearing). The email agent consumes it server-side, with nobody holding an HTTP connection.
- **`finalize` on failure** (worker unreachable, or the stream ended without `complete`): mark every upload in the batch `failed` with the error and close the session `completed_with_errors`. **Uploads are never left in `processing`.** Then throw, so the caller's retry or reprocess logic takes over.
- **`finalize` on success:** merge this file's sections into the record's merged extraction (all files on the record combined), update the record's identity (legal name, tax ID) from the documents, and **auto-map** right away (§6.1).
- After extraction, copy the worker's per-file verdict (`insufficient`, unreadable) back onto the caller's own row, so callers can report per file ("acord.pdf was read; photo.png isn't an insurance document").

### 6.1 Auto-map on extraction

Mapping to the record's data model (pick the record type, then map extracted fields to its field definitions) runs **as part of finalize**. It doesn't wait until someone opens a review screen. Every reader of "is this record ready?" (list views, readiness checks, the email agent's reply) sees the same current state. The result is `ready` (no missing mandatory, invalid or conflicting fields) or `needs_info`, computed by the same function the review screen uses so the two can never disagree.

---

## 7. Database Schema (MongoDB)

```js
// document_extractions — the result cache AND the canonical extraction record
{
  documentHash,               // sha256 of file bytes (indexed) — the cache key
  tenantId, userId,
  documentUploadId,           // first upload
  documentUploadIds: [String],// every upload that resolved to this content
  originalFileName, fileSize,
  ocrSource,                  // mistral | cache | native
  ocrMarkdown,                // page-delimited OCR text (layer-1 cache)
  pageText, sectionPageRanges,
  detectedTypes: [String], isMultiType,
  finalExtraction,            // { acord: {...}, loss_run: {...}, generic: {...}, document_metadata: { extraction_status, page_ranges } }
  extractionModel, extractionTemperature /* 0 */, processingTime,
  promptVersion, schemaVersion, extractionVersion,   // layer-2 cache validity
  auditData, lastAccessed, accessCount
}

// document_uploads — one per received file, per record
{ applicationId, tenantId, fileName, s3Url, documentHash, source,
  extractionStatus /* processing | completed | failed */, extractionError, extractionNote }

// submission_sessions — one per intake batch
{ applicationId, tenantId, userId, batchStatus /* processing | completed_success | completed_with_errors */ }
```

Store `tenantId` with **one type across both services**. If the API writes an `ObjectId` and the worker writes a `string`, tenant-filtered queries miss rows until you query both forms.

---

## 8. Cost & Performance Controls

- **Two cache layers** (§4.2). The OCR layer survives prompt changes. Resends, forwards and re-uploads cost nothing.
- **Page-range scoping** (§4.5). Each extractor sees only its pages, and the budget follows the page count.
- **Concurrency caps** at request level and process level, matched to the provider's rate limit.
- **Selective rasterization** on the native engine: only scanned, garbled or tabular pages are sent as images.
- **Per-call usage log:** every model call writes a row (`feature` = `document_ocr` | `document_classification` | `document_extraction:<type>`, model, tokens, pages, cost from a price table). Spend can be attributed per document type and per tenant.
- **Deterministic before model:** the rule-based page classifier and domain-content gate run without a model call, and the raw-text attachment for non-domain documents is free.

---

## 9. Integrations & Environment

| Service | Use | Env |
|---------|-----|-----|
| **Mistral** (OCR + chat) | Page OCR to Markdown, section JSON extraction, optional LLM page classification | `MISTRAL_API_KEY`, `MISTRAL_API_BASE`, `MISTRAL_OCR_MODEL`, `MISTRAL_EXTRACTION_MODEL`, `MISTRAL_OCR_MAX_RETRIES`, `MISTRAL_OCR_RETRY_BASE_DELAY`, `MISTRAL_SEND_PAGE_IMAGES` |
| **OCR tuning** | Engine and rasterization heuristics | `PDF_ENGINE` (`mistral_ocr` \| `native`), `OCR_MIN_TEXT_CHARS`, `OCR_MIN_ALNUM_RATIO`, `OCR_IMAGE_FOR_TABLES`, `OCR_ALWAYS_IMAGE`, `OCR_RENDER_DPI` |
| **Concurrency** | Request and process limits | `MAX_CONCURRENT_STREAMS`, `MAX_CONCURRENT_MISTRAL_CALLS`, `EXTRACTION_MAX_CONCURRENCY` |
| **Worker** | FastAPI service | `EXTRACTION_HOST`, `EXTRACTION_PORT`, `UPLOAD_MAX_SIZE`, `ALLOWED_FILE_TYPES` |
| **MongoDB** | Shared result store | `MONGODB_URI`, `MONGODB_DB_NAME` |
| **API → worker** | Where intake posts | `PYTHON_BACKEND_URL` |
| **Object storage** | Original files | `STORAGE_TYPE` plus that backend's credentials |

Python dependencies: `fastapi`, `uvicorn`, `PyMuPDF` (image → PDF, page render), `PyPDF2` (native text), `aiohttp` (streaming model calls), `pymongo`, `pyyaml`. Node: `form-data`, `axios` (stream response), `readline`.

---

## 10. Pitfalls (learned in production)

| Symptom | Cause | Fix |
|---------|-------|-----|
| Loss-run values attached to the wrong claim rows | The native text layer linearizes tables out of row/column order | Detect numeric grids and render those pages as images for the vision model |
| A scanned page came back "empty" with no error | The page had a text layer full of mojibake, so the length check passed | Add an alphanumeric-ratio check, and rasterize below it |
| Prompt fix shipped, old answers kept coming back | The result cache had no version key | Version the prompt, schema and extraction logic, and make the final cache valid only when all three match |
| Uploads stuck "processing" forever | The worker crashed before sending `complete`, and the consumer treated stream close as success | Close without `complete` = failure. Mark every upload in the batch failed. |
| The UI got 404 fetching a "complete" extraction | The save was fire-and-forget, so `complete` went out before the write landed | Await the save, then emit `complete` |
| A children's book was "extracted" as an empty application | Schema-bound prompts correctly return almost nothing for off-domain content | Domain-content gate → `insufficient`, plus raw page text kept for viewing |
| `.md` files routed to different extractors on different machines | `mimetypes.guess_type` depends on the host (registry on Windows) | Register the MIME types you depend on explicitly at startup |
| 429s under load | Per-request parallelism multiplied by concurrent uploads | Add a process-wide semaphore on model calls in addition to the per-request one |

---

## 11. Porting Checklist

**Intake**
- [ ] SHA-256 every file; tenant-scoped duplicate check; link byte-identical, already-extracted files without calling the worker
- [ ] Upload row per file + session per batch, with `source`; cap by bytes, not pages
- [ ] One file per worker call; multipart form with ids, manifest and force flags

**Worker**
- [ ] Normalize images to PDF; always clean up temp files
- [ ] Layered cache: OCR by `(hash, tenant)`; final result by `(hash, tenant)` + prompt/schema/extraction versions; force flags purge or bypass
- [ ] Replay cache hits as the normal event stream
- [ ] OCR engine switch: OCR model → page-delimited Markdown, or native text + selective rasterization (scanned / garbled / tabular)
- [ ] Page classifier → typed, possibly non-contiguous page ranges; every page in exactly one section; generic lowest priority
- [ ] Domain-content gate → `insufficient` + raw page text, with no extra model call
- [ ] One prompt and schema per section type; send only that section's pages
- [ ] Per-request + process-wide concurrency limits; token budget and timeout scaled by page count
- [ ] Token streaming with an incremental top-level-key JSON parser; row events for tables
- [ ] JSON repair (extract, close truncation, fix quotes, strip citations); retry only the failed section
- [ ] Domain validators → `__validated__` chunk
- [ ] Honest status (`completed` / `partial_success` / `partial_failed` / `failed` / `insufficient`)
- [ ] Persist (versions, models, counts, OCR text) **before** emitting `complete`

**Consumer**
- [ ] NDJSON consumer: accumulate sub-chunks, replace on `__validated__`; `complete` = success even after `error`; close without `complete` = failure
- [ ] Finalize: on failure, mark uploads failed (never stranded); on success, merge into the record, update identity, auto-map to the data model
- [ ] Portal proxies the stream for live progress; background consumers use the same protocol
- [ ] Per-call usage and cost logging, tagged by feature and section type

---

<!-- Part of the ShipFactory feature spec library — https://github.com/vishalquantana/shipfactory -->

## About Us

We are [Quantana](https://quantana.com.au), an AI-first design and development agency working with Fortune 500s to build bespoke AI solutions and provide the audit and training needed to ensure success. [Click here to learn more](https://quantana.com.au).

## License

MIT
