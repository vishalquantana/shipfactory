# E-Signature Documents (Self-Hosted DocuSign)

## Feature Specification for Porting

This document describes a self-hosted, DocuSign-style electronic-signature module: upload a PDF, drag signature/initials/date/text fields onto it, assign them to ordered signers, send token-based signing links by email, capture signatures (type / draw / upload), flatten everything onto the PDF with a Certificate of Completion, and maintain a tamper-evident, hash-chained audit trail with a public signing-trail page.

It also bundles two client-side PDF utilities that ship alongside the signing workflow in the same "Docs" hub: **Compress** and **Merge**.

> In the source product this module is surfaced to users as **"Docs"** but is named **"Contracts"** internally (`contracts` table, `routers/contracts.py`). The two names refer to the same feature.

---

## 1. Overview

A document module with three tabs:

- **Documents (signing)** — the core e-signature workflow
- **Compress** — reduce a PDF's file size, fully in-browser
- **Merge** — combine/reorder pages from multiple PDFs, fully in-browser

The signing workflow supports:

- PDF upload with scanned-document detection (warns when a PDF has no extractable text)
- A drag-to-place field builder (signature, initials, date, text) positioned as page-relative percentages
- Multiple ordered signers with per-signer field assignment, plus CC recipients (notified, no signing authority)
- Sequential signing batches (lower-order signers must finish before the next batch is emailed)
- Token-based signing links (hashed, 30-day expiry), email delivery, manual resend, and an automatic reminder scheduler
- Three signature input methods: type (rendered in a cursive font), draw (canvas), or upload an image
- Server-side flattening of all field values onto the PDF + an appended **Certificate of Completion** page
- Void, duplicate, copy-shareable-link, and delete (draft only)
- A **tamper-evident audit log** (SHA-256 hash chain) and a public signing-trail page protected by magic link + optional OTP

Statuses: `draft → sent → partial → completed`, plus `voided`.

---

## 2. User Flow

### 2.1 Sender flow

```
Docs → "New Document"
        │
        ▼
┌────────────────────────────────────────────┐
│            Document Editor                  │
│                                             │
│  [ Upload PDF ]   (warns if scanned/no text)│
│                                             │
│  ┌───────────────────────────────────────┐ │
│  │  PDF page with draggable fields       │ │
│  │   ┌─────────┐                         │ │
│  │   │ Signature│  ← drag to place,       │ │
│  │   └─────────┘     resize, assign signer│ │
│  └───────────────────────────────────────┘ │
│                                             │
│  Signers:  1. Alice (order 1)  [color]      │
│            2. Bob   (order 2)  [color]      │
│  CC:       finance@acme.com                 │
│  Field types: Signature │ Initials │ Date │ │
│               Text                          │
│  Personal message: "Please countersign…"    │
│  Linked deal: Acme Corp (optional)          │
│                                             │
│  [ Save Draft ]        [ Send for Signing ] │
└────────────────────────────────────────────┘
        │  send
        ▼
  Tokens generated, emails sent to order-1 signers
  status = sent, audit: "sent"
        │
        ▼
  Track on Document Detail page:
   • per-signer status (pending/viewed/signed/declined)
   • copy/regenerate signing link, resend, void
   • download original / signed PDF
   • audit log with hash-chain verification
```

### 2.2 Signer flow (public, no account)

```
Email "Please sign: {title}"  →  click signing link (token)
        │
        ▼
┌────────────────────────────────────────────┐
│            Signing Page                     │
│  (audit: "viewed" on open — records IP/UA)  │
│                                             │
│  ┌───────────────────────────────────────┐ │
│  │  PDF with YOUR fields highlighted      │ │
│  │   ┌─────────┐                         │ │
│  │   │ click to │  → Signature Capture:   │ │
│  │   │  sign    │     Type / Draw / Upload│ │
│  │   └─────────┘                         │ │
│  │   Date fields auto-fill; text inputs   │ │
│  └───────────────────────────────────────┘ │
│                                             │
│  [ Decline (with reason) ]   [ Submit ]     │
└────────────────────────────────────────────┘
        │ submit (audit: "signed", IP/UA)
        ▼
  All required fields filled → signer = signed
        │
        ├─ more signers remain → next batch emailed (status=partial)
        │
        └─ all signed → status=completed
              • flatten field values onto PDF
              • append Certificate of Completion
              • upload signed.pdf to S3
              • email completed PDF to signers + CC
              • audit: "completed"
```

---

## 3. Frontend Components

File tree (React, plain inline styles — no UI framework assumed):

```
components/
  DocsPage.jsx          # tab router: Documents | Compress | Merge
  ContractsPage.jsx     # documents list
  ContractEditor.jsx    # create/edit draft, field builder
  DocDetailPage.jsx     # detail / tracking / audit (also under pages/)
  PdfViewer.jsx         # reusable pdf.js renderer with field overlays
  SignatureCapture.jsx  # type/draw/upload signature modal
  DocCompress.jsx       # Compress tab
  DocMerge.jsx          # Merge tab
pages/
  SigningPage.jsx       # public signer-facing page
  DocDetailPage.jsx     # document detail + audit + actions
lib/
  pdfCompress.js        # client-side compression engine
  pdfMerge.js           # client-side merge engine
```

### 3.1 Docs Hub (`DocsPage.jsx`)

| Aspect | Detail |
|--------|--------|
| **Tabs** | "All" (documents list), "Compress", "Merge" |
| **Loading** | Child components are lazy-loaded |
| **Purpose** | Single entry point that routes between the signing workflow and the two PDF utilities |

### 3.2 Documents List (`ContractsPage.jsx`)

| Aspect | Detail |
|--------|--------|
| **Rows** | Title, status badge, signer progress (`X/Y signed`), created date |
| **Filters** | Status filter (draft / sent / partial / completed / voided), search by title or linked deal |
| **Actions** | "New Document", open editor (draft) or detail (sent+) |

### 3.3 Document Editor (`ContractEditor.jsx`)

| Aspect | Detail |
|--------|--------|
| **PDF upload** | File input → backend `upload-pdf`. Detects scanned/no-text PDFs and warns (fields can't anchor to text, but placement still works) |
| **Field builder** | Drag to place fields on the rendered PDF; resize; each field has a type and an assigned signer (color-coded) |
| **Field types** | `signature`, `initials`, `date`, `text`; `required` toggle |
| **Field geometry** | Stored as page-relative percentages (`x_pct`, `y_pct`, `w_pct`, `h_pct`, `page`) so they survive different render scales |
| **Signers** | Add/remove/reorder; name + email + signing `order`. Frontend assigns temporary IDs that the backend maps to real UUIDs on save |
| **CC** | Name + email; receive the completed PDF only |
| **Extras** | Personal message, optional linked deal/record dropdown |
| **Save** | `PATCH` draft (title, message, fields, signers, CCs); "Send for Signing" validates ≥1 signer and that every required field is assigned |

### 3.4 PDF Viewer (`PdfViewer.jsx`)

| Aspect | Detail |
|--------|--------|
| **Renderer** | `pdfjs-dist` (v4.4.168) |
| **Features** | Zoom, multi-page, absolute-positioned field overlays driven by the percentage geometry |
| **Reuse** | Shared by editor, detail page, and signing page |

### 3.5 Signature Capture (`SignatureCapture.jsx`)

| Aspect | Detail |
|--------|--------|
| **Type** | Type a name → rendered to a cursive-font PNG (server has the same font for flattening) |
| **Draw** | Canvas drawing via `react-signature-canvas` (v1.0.6) |
| **Upload** | Upload a signature image |
| **Output** | Base64 PNG stored as the field `value` |

### 3.6 Signing Page (`SigningPage.jsx`) — public

| Aspect | Detail |
|--------|--------|
| **Auth** | None — gated solely by the URL token |
| **On open** | Calls `POST /api/sign/{token}/viewed` (records IP + user-agent) |
| **Rendering** | Shows the PDF with only this signer's fields highlighted/editable |
| **Inputs** | Signature/initials via Signature Capture; date fields auto-fill; text fields are inline inputs |
| **Decline** | Decline with a reason |
| **Submit** | Validates required fields, submits values; session ends on completion |

### 3.7 Document Detail / Audit (`pages/DocDetailPage.jsx`)

| Aspect | Detail |
|--------|--------|
| **Status** | Per-signer status tracker; overall status |
| **Actions** | Send/resend, void (with reason), duplicate, copy/regenerate signing link, download original/signed PDF, delete (draft only) |
| **Audit log** | Renders the full event log and **verifies the hash chain client-side** (`verifyChain()`): recomputes each entry's hash from the previous entry's hash and flags any break |

### 3.8 Compress Tab (`DocCompress.jsx`)

| Aspect | Detail |
|--------|--------|
| **Upload** | Drag-and-drop zone, **max 100 MB** |
| **Quality levels** | "Email-ready" (max dim 1000px), "Balanced" (1600px), "High quality" (2200px) |
| **Preview** | Canvas preview of page 1; live estimate of savings (image-byte breakdown) |
| **Progress** | Phased progress bar: loading → scanning → compressing image N of total → saving |
| **Result** | Original size, compressed size, savings %; download as `{name}-compressed.pdf` |
| **Processing** | 100% client-side (see §9) |

### 3.9 Merge Tab (`DocMerge.jsx`)

| Aspect | Detail |
|--------|--------|
| **Upload** | Multi-file drop zone; color-coded legend per source file with page counts |
| **Reorder** | Drag-and-drop thumbnail grid (`@hello-pangea/dnd`); thumbnails rendered at ~0.3 scale, JPEG q0.6 |
| **Select** | Per-page checkbox to include/exclude; X to remove a page |
| **Option** | "Also compress output" applies Balanced compression to the merged result |
| **Output** | Download `merged.pdf` |
| **Processing** | 100% client-side (see §9) |

---

## 4. Frontend Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `pdfjs-dist` | 4.4.168 | PDF rendering, page previews, text extraction (scanned-PDF detection) |
| `react-signature-canvas` | 1.0.6 | Canvas-based signature drawing |
| `pdf-lib` | ^1.17.1 | Client-side PDF structure manipulation (compress + merge) |
| `@hello-pangea/dnd` | 17.0.0 | Drag-and-drop page reordering in the Merge tab |

Browser APIs used directly: `CompressionStream` / `DecompressionStream` (zlib), Canvas / `OffscreenCanvas`, `createImageBitmap`.

---

## 5. Backend API

Router: `routers/contracts.py`. All authenticated routes are workspace-scoped (Bearer token). Public signing/audit routes are gated by hashed tokens or OTP cookies — never by session auth.

### 5.1 Authenticated — `/api/contracts`

| Method & Path | Purpose |
|---------------|---------|
| `POST /upload-pdf` | Upload a PDF (max 50 MB) to S3. Runs text extraction; returns `{ url, warning? }` where `warning` flags a scanned/no-text PDF |
| `GET /pdf-proxy?url=` | Auth-gated proxy for S3 PDF bytes (CORS workaround for the viewer) |
| `POST /` | Create a contract: `{ title, original_pdf_url, signed_pdf_url?, deal_id? }` → `{ id, status }` |
| `GET /` | List contracts in the workspace with signer counts and linked-deal title; optional `deal_id` filter |
| `GET /{id}` | Full detail: signers, fields, CCs, audit log |
| `PATCH /{id}` | Update a **draft**: title, personal_message, fields, signers, CCs. Maps temp signer IDs → UUIDs |
| `POST /{id}/send` | Validate ≥1 signer + all required fields assigned; generate hashed signing tokens (30-day expiry); email only the lowest-order signer batch; status → `sent`; audit `sent` |
| `POST /{id}/void` | Cancel signing, nullify tokens, notify recipients; audit `voided` (+ reason) |
| `POST /{id}/resend/{signer_id}` | Regenerate that signer's token and re-email; audit `resent` |
| `POST /{id}/duplicate` | Clone to a new draft (copies fields/signers/CCs, resets values + tokens, keeps the PDF URL) → new id |
| `POST /{id}/signers/{signer_id}/link` | Regenerate a signing link and return the URL **without** emailing (copy-paste sharing); audit `link_regenerated` |
| `DELETE /{id}` | Delete a **draft only**; cascade signers, fields, CCs, audit rows |

### 5.2 Public signing — `/api/sign`

| Method & Path | Purpose |
|---------------|---------|
| `GET /{token}/pdf` | Fetch the PDF for the signing page (token-gated, no auth) |
| `GET /{token}` | Signing view data: title + this signer's editable fields only |
| `POST /{token}/viewed` | Mark viewed; record IP + user-agent; audit `viewed` |
| `POST /{token}` | Submit signature values. `SELECT … FOR UPDATE` lock on the signer row (race guard); validate required fields; set signer `signed`, record IP/UA; advance to next batch or set `completed`; on completion, flatten PDF + email; audit `signed` (and `completed`) |
| `POST /{token}/decline` | Set signer `declined` with reason, record IP/UA; audit `declined` |

### 5.3 Public audit — `/api/sign/audit`

| Method & Path | Purpose |
|---------------|---------|
| `GET /audit/{id}` | Signing-trail page data. **Tri-path auth:** workspace member **OR** valid magic-link token **OR** audit-access cookie. Returns signers, CCs, full audit log + hash chain |
| `GET /audit/{id}/pdf` | Download the signed PDF from the audit page |
| `POST /audit/{id}/access` | Email an OTP to a known recipient to unlock the audit page |
| `POST /audit/{id}/verify` | Verify the OTP and set an audit-access cookie |

---

## 6. Database Schema (PostgreSQL)

```sql
-- Documents / contracts
CREATE TABLE public.contracts (
    id                  uuid DEFAULT gen_random_uuid() NOT NULL,
    deal_id             uuid,                      -- optional FK (ON DELETE CASCADE)
    workspace_id        uuid NOT NULL,
    title               text NOT NULL,
    status              text DEFAULT 'draft' NOT NULL,
        -- CHECK: draft | sent | partial | completed | voided
    original_pdf_url    text NOT NULL,             -- s3://bucket/key (legacy: https://)
    signed_pdf_url      text,                      -- set after all signers complete
    voided_at           timestamptz,
    voided_by           uuid,
    void_reason         text,
    previous_version_id uuid,                      -- set on duplicate
    personal_message    text,
    created_by          uuid NOT NULL,
    created_at          timestamptz DEFAULT now() NOT NULL,
    updated_at          timestamptz DEFAULT now() NOT NULL
);

-- Ordered signers
CREATE TABLE public.contract_signers (
    id                 uuid DEFAULT gen_random_uuid() NOT NULL,
    contract_id        uuid NOT NULL,
    workspace_id       uuid NOT NULL,
    name               text NOT NULL,
    email              text NOT NULL,
    "order"            integer DEFAULT 1 NOT NULL,  -- signing sequence / batch
    signing_token_hash text NOT NULL,              -- SHA-256 of the token (raw token never stored)
    token_expires_at   timestamptz NOT NULL,       -- 30-day default
    status             text DEFAULT 'pending' NOT NULL,
        -- CHECK: pending | viewed | signed | declined
    signed_at          timestamptz,
    declined_at        timestamptz,
    decline_reason     text,
    ip_address         text,                       -- x-forwarded-for or client.host
    user_agent         text
);

-- Fields placed on the PDF
CREATE TABLE public.contract_fields (
    id           uuid DEFAULT gen_random_uuid() NOT NULL,
    contract_id  uuid NOT NULL,
    workspace_id uuid NOT NULL,
    signer_id    uuid,                              -- FK to contract_signers
    type         text NOT NULL,                     -- signature | initials | date | text
    page         integer NOT NULL,
    x_pct        double precision NOT NULL,         -- 0..1 page-relative position
    y_pct        double precision NOT NULL,
    w_pct        double precision NOT NULL,         -- 0..1 page-relative size
    h_pct        double precision NOT NULL,
    label        text,
    required     boolean DEFAULT true NOT NULL,
    value        text                               -- base64 PNG (sig/initials) or text
);

-- CC recipients (completion email only)
CREATE TABLE public.contract_cc (
    id          uuid DEFAULT gen_random_uuid() NOT NULL,
    contract_id uuid NOT NULL,
    name        text NOT NULL,
    email       text NOT NULL
);

-- Tamper-evident audit log (hash chain)
CREATE TABLE public.contract_audit_log (
    id              uuid DEFAULT gen_random_uuid() NOT NULL,
    contract_id     uuid NOT NULL,
    workspace_id    uuid NOT NULL,
    event           text NOT NULL,
        -- created | sent | viewed | signed | declined | completed
        -- | voided | resent | reminded | link_regenerated
    actor_email     text,
    actor_signer_id uuid,
    ip_address      text,
    user_agent      text,
    metadata        jsonb,                          -- event-specific (reason, recipient_count, …)
    occurred_at     timestamptz DEFAULT now() NOT NULL,
    prev_hash       text,                           -- SHA-256 of the previous entry
    hash            text NOT NULL                   -- SHA-256 of this entry (see §8)
);

-- Magic-link tokens for the public audit page
CREATE TABLE public.contract_audit_tokens (
    id          uuid DEFAULT gen_random_uuid() PRIMARY KEY,
    contract_id uuid NOT NULL,
    email       text NOT NULL,
    token_hash  text NOT NULL,                      -- SHA-256
    expires_at  timestamptz NOT NULL,               -- 90-day default
    created_at  timestamptz DEFAULT now() NOT NULL
);
```

---

## 7. Integrations

### 7.1 S3 storage (`boto3`)
- **Uploads:** `contracts/{contract_id}/uploads/{uuid}/{filename}`
- **Signed output:** `contracts/{contract_id}/signed.pdf`
- URLs stored as `s3://bucket/key` (legacy rows may hold `https://`); the viewer fetches via `GET /api/contracts/pdf-proxy?url=` to dodge CORS.

### 7.2 Email (SendGrid) — `services/email.py`
| Function | When |
|----------|------|
| `send_contract_signing_request()` | Initial signing request to the active batch |
| `send_contract_reminder()` | Manual resend + automatic reminders |
| `send_contract_completion()` | All signers done — notifies signers + CC, attaches the signed PDF |
| `send_contract_void_notification()` | On void |
| `send_contract_decline_notification()` | On decline |

Configured via `SENDGRID_API_KEY`.

### 7.3 PDF processing — `services/pdf_processor.py`
- `render_typed_signature_png()` — renders a typed name to a cursive PNG (font: **Dancing Script**; ship the `.ttf` so server output matches the client preview).
- `flatten_signatures()` — stamps every field value onto the PDF at its percentage geometry, then appends a **Certificate of Completion** page (signer names, emails, timestamps, IP/UA, document hash).
- `compute_audit_hash()` — the SHA-256 chain hash (see §8).
- Libraries: `pypdf` (manipulation), `reportlab` (Certificate page), `Pillow` (image rendering).

### 7.4 Automatic reminder scheduler — `services/contract_reminders.py`
- Runs on the app's startup scheduler (cron-like loop in `main.py`).
- For each pending signer in the **active batch** of a `sent`/`partial` contract: if fewer than **3** reminders sent **and** >3 days since the last notification → regenerate token, send a reminder, audit `reminded`.

---

## 8. Audit Trail & Chain of Custody

Every state change calls a single `_write_audit()` helper that appends one row to `contract_audit_log`.

**Events:** `created`, `sent`, `viewed`, `signed`, `declined`, `completed`, `voided`, `resent`, `reminded`, `link_regenerated`.

**Captured per entry:** `event`, `actor_email`, `actor_signer_id` (for signer-scoped events), `ip_address` (from `x-forwarded-for` or `request.client.host`), `user_agent`, `metadata` (jsonb), `occurred_at`.

**Hash chain:** each row stores `prev_hash` (the previous entry's `hash`) and its own `hash`, computed as a SHA-256 over the concatenation of the entry's identifying fields plus the previous hash:

```
hash = SHA256( row_id ‖ contract_id ‖ prev_hash ‖ event ‖ actor_email ‖ occurred_at ‖ metadata )
```

Because each hash folds in the previous one, altering or deleting any historical entry invalidates every subsequent hash. The frontend `verifyChain()` recomputes the chain and flags any break, so tampering is visibly detectable on the signing-trail page.

**Audit page access (defense in depth):**
1. Workspace member (session), **or**
2. A 90-day magic-link token emailed to each signer/CC (`contract_audit_tokens`), **or**
3. An OTP flow (`/access` → `/verify`) that sets an access cookie for return visits.

---

## 9. PDF Utilities (client-side)

Both tools run **entirely in the browser** — no backend endpoints, no server bandwidth/CPU. Files never leave the device.

### 9.1 Compress — `lib/pdfCompress.js`
1. **Scan:** enumerate indirect objects, find image XObjects > 200 bytes and > 400 px; classify `DCTDecode` (JPEG) vs `FlateDecode` (raw/PNG).
2. **Estimate:** `estimateCompression()` returns original size + image-byte breakdown; projects output per quality level (Email ≈ keep 20% of image bytes, Balanced ≈ 40%, High ≈ 60%).
3. **Recompress each image** (`recompressImage()`):
   - Decode (`createImageBitmap` for JPEG; inflate for Flate). Apply **PNG predictor** unfilter for `Predictor ≥ 10` (filters 0–4: None/Sub/Up/Average/Paeth).
   - Resolve color space: Indexed→palette expansion (RGB & CMYK base, CMYK→RGB), plus Gray/RGB/CMYK.
   - Downsample to the level's `maxDim` (preserve aspect), re-encode JPEG at q ≈ 0.42–0.78.
   - **SMask (alpha):** decode, resize to match, re-deflate, re-attach.
   - **Skip-on-failure:** if an image can't decode or doesn't shrink, keep the original (never bubble an error).
4. **Save:** replace image streams, strip metadata (Title/Author/Producer/Creator), write with `useObjectStreams: true`. Returns bytes + stats (`originalSize`, `compressedSize`, `imagesProcessed`, …).
   - **Input limit:** 100 MB. Image min: > 200 bytes & > 400 px.

### 9.2 Merge — `lib/pdfMerge.js`
- `mergePdfs(order, sourceBuffers)`: `order` is an array of `{ fileIndex, pageIndex }` in the desired sequence; loads all sources in parallel, `copyPages()` + `addPage()` per reference, saves with `useObjectStreams: true`. Honors drag-reordering and page subsetting. Optionally pipes the result through `compressPdf()` ("Also compress output"). No hard file/size cap (browser-memory bound).

---

## 10. Data Collected

| Data Point | Source | Auto/Manual |
|------------|--------|-------------|
| Document title, personal message | Sender input | Manual |
| Original PDF | Sender upload | Manual |
| Field geometry + types | Sender (drag builder) | Manual |
| Signers (name/email/order), CCs | Sender input | Manual |
| Signature/initials images, text values | Signer input | Manual |
| Signing/decline timestamps | Server `now()` | Auto |
| Signer IP address | `x-forwarded-for` / `client.host` | Auto |
| Signer user-agent | Request header | Auto |
| Audit events + hash chain | Server on each state change | Auto |
| Signed PDF + Certificate of Completion | Server flatten | Auto |
| Workspace / created_by | Auth session | Auto |

---

## 11. Porting Checklist

To replicate this module in another application:

- [ ] **PDF upload + storage** — upload endpoint to S3 (or equivalent); detect scanned/no-text PDFs and warn
- [ ] **CORS-safe viewer** — render with `pdfjs-dist`; proxy object-store bytes through an auth-gated endpoint
- [ ] **Field builder** — drag-to-place signature/initials/date/text fields; store geometry as page-relative percentages (`x/y/w/h_pct`, `page`)
- [ ] **Signers + ordering** — per-signer field assignment, signing `order`/batches, CC recipients
- [ ] **Tokenized links** — generate a random token, store only its **SHA-256 hash** with an expiry; gate public pages by token, never by session
- [ ] **Signature capture** — type (font-rendered PNG), draw (`react-signature-canvas`), and image upload; store as base64 PNG
- [ ] **Send + sequential batches** — email only the active batch; advance when the batch completes
- [ ] **Race guard** — `SELECT … FOR UPDATE` on the signer row when accepting a submission
- [ ] **Flatten + certificate** — stamp values onto the PDF (`pypdf`/`Pillow`) and append a Certificate of Completion (`reportlab`); ship the cursive font on the server
- [ ] **Lifecycle actions** — resend, void (with reason), duplicate-to-draft, copy/regenerate link, delete (draft only)
- [ ] **Reminder scheduler** — auto-remind pending signers (cap 3, ≥3-day spacing) on a startup cron loop
- [ ] **Tamper-evident audit log** — append-only table with `prev_hash` + `hash` SHA-256 chain; record event, actor, IP, user-agent, metadata
- [ ] **Chain verification UI** — recompute and verify the hash chain client-side; flag any break
- [ ] **Public audit page** — tri-path access (member / 90-day magic link / OTP cookie)
- [ ] **Email templates** — signing request, reminder, completion (attach signed PDF), void, decline
- [ ] **Status model** — `draft → sent → partial → completed`, plus `voided`
- [ ] **(Optional) Compress utility** — client-side image downsample + JPEG re-encode with PNG-predictor & SMask handling, skip-on-failure (`pdf-lib` + Canvas + `CompressionStream`)
- [ ] **(Optional) Merge utility** — client-side page reorder/subset across multiple PDFs (`pdf-lib` + `@hello-pangea/dnd`), optional compress-on-output

---

<!-- Part of the ShipFactory feature spec library — https://github.com/vishalquantana/shipfactory -->

## About Us

We are [Quantana](https://quantana.com.au), an AI-first design and development agency working with Fortune 500s to build bespoke AI solutions and provide the audit and training needed to ensure success. [Click here to learn more](https://quantana.com.au).

## License

MIT
