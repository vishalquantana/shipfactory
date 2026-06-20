# Multi-Source CSV Ingestion Pipeline

> A portable pipeline for ingesting CSV exports from **multiple upstream platforms** that each use different header conventions. It auto-detects which source a file came from by sniffing header signatures, maps that source's columns onto a unified schema, coerces raw string cells into typed values, batch-inserts them into Postgres, and optionally archives the raw file to S3-compatible object storage. Re-usable in any SaaS that imports tabular data from third parties you don't control.

---

## 1. Overview

Many SaaS products must ingest CSV exports from external platforms ("Source A", "Source B", "Source C", …). The pain: every source names its columns differently — one uses `Title Case`, another `snake_case`, another `UPPER_SNAKE_CASE`; some prepend metadata preamble rows before the real header; the same logical field ("revenue") is called `Revenue`, `gmv`, or `Revenue (GMV)` depending on the source.

This pipeline solves that with four decoupled stages:

1. **Source detection** — sniff the header row to figure out which platform/report a file is, with zero user input.
2. **Column mapping** — a per-source registry that maps each source's column header → your canonical DB column name.
3. **Type coercion** — convert raw string cells to numbers/strings/null, respecting a per-column allow-list of "always-string" fields.
4. **Batch insert + archive** — chunked inserts (with optional dedupe), then archive the untouched original to object storage for replay/audit.

Two flavors ship in the reference implementation:

- **Simple variant** — caller passes the source explicitly (`platform` form field); a flat column map is applied. See `parseAndInsertCSV`.
- **Auto-detect variant** — source is sniffed from header signatures and routed to one of N tables. See `parseAndInsertReport` + `detectReportType`.

This doc abstracts the reference domain (ad-platform reports for a `brand`) into generic placeholders. Replace `<YourApp>`, `Source A/B/C`, and the canonical column set with your own.

---

## 2. Pipeline Flow

```
  ┌─────────────┐
  │  HTTP POST  │  multipart/form-data: file + (optional) source hint
  │  /upload    │
  └──────┬──────┘
         │  file.text()
         ▼
  ┌──────────────────┐
  │ 1. DETECT SOURCE │  sniff header signatures → which platform/report?
  │  detect.ts       │  (skip preamble rows, find real header row)
  └──────┬───────────┘
         │  reportType + headerRowIndex + dataStartIndex
         ▼
  ┌──────────────────┐
  │ 2. MAP COLUMNS   │  COLUMN_MAPS[source]: "Revenue (GMV)" → revenue
  │  column-maps.ts  │
  └──────┬───────────┘
         │  csvCol → dbCol
         ▼
  ┌──────────────────┐
  │ 3. COERCE TYPES  │  "1,234" → 1234 ; "NA" → null ; STRING_COLUMNS stay text
  │  parseValue()    │
  └──────┬───────────┘
         │  typed row objects
         ▼
  ┌──────────────────┐
  │ 4. BATCH INSERT  │  chunks of 500, onConflictDoNothing dedupe
  │  db.insert()     │
  └──────┬───────────┘
         │
         ▼
  ┌──────────────────┐
  │ 5. ARCHIVE (opt) │  PutObject raw CSV → uploads/<tenant>/<source>/<ts>-<name>
  │  uploadToS3()    │
  └──────┬───────────┘
         │
         ▼
   { rowsInserted, rawDataS3Key }
```

Stages 1–3 are pure functions (no I/O) and trivially unit-testable. Stages 4–5 are the only ones that touch external systems.

---

## 3. Source Detection

The detector receives the raw file split into lines and answers three questions: **which source is this**, **which line is the real header**, and **where does data start**. Two header conventions are handled: a "clean" format whose header is line 1, and a "preamble" format with metadata rows before the header.

Detection is signature-based: each source is identified by the *presence* (and sometimes *absence*) of marker columns in the header row. This is robust to column reordering and added columns.

```typescript
// detect.ts

// Clean format: header is line 1, no preamble
const KEYWORD_REPORT_SIGNATURE = ["Keyword", "Keyword Type", "Sponsored SOV %"];

// Each rule recognizes one source/report by header markers (presence + absence)
const DETECTION_RULES: Array<{ check: (headers: string[]) => boolean; type: ReportType }> = [
  { check: (h) => h.includes("CAMPAIGN_UPTIME_PERCENTAGE"), type: "im_campaign_daily" },
  { check: (h) => h.includes("SEARCH_QUERY"), type: "im_search_query" },
  { check: (h) => h.includes("CITY") && h.includes("PRODUCT_NAME") && h.includes("METRICS_DATE"), type: "im_granular" },
  { check: (h) => h.includes("AD_PROPERTY") && h.includes("KEYWORD") && !h.includes("METRICS_DATE"), type: "im_placement" },
  { check: (h) => h.includes("PRODUCT_ID"), type: "im_campaign_product" },
  { check: (h) => h.includes("CITY") && h.includes("AD_PROPERTY_COUNT") && !h.includes("METRICS_DATE") && !h.includes("PRODUCT_NAME"), type: "im_campaign_city" },
  { check: (h) => h.includes("CAMPAIGN_ID") && h.includes("TOTAL_ROI") && !h.includes("METRICS_DATE") && !h.includes("CAMPAIGN_UPTIME_PERCENTAGE"), type: "im_summary" },
];

export function detectReportType(lines: string[]): DetectionResult {
  if (lines.length === 0) {
    return { reportType: null, headerRowIndex: -1, dataStartIndex: -1, headers: [], error: "File is empty" };
  }

  // Case A — clean format: header on line 1
  const firstLineCols = parseCsvLine(lines[0]);
  if (KEYWORD_REPORT_SIGNATURE.every((sig) => firstLineCols.includes(sig))) {
    return { reportType: "im_keyword_report", headerRowIndex: 0, dataStartIndex: 1, headers: firstLineCols };
  }

  // Case B — preamble format: line 1 is a known sentinel, header is further down
  if (firstLineCols[0]?.trim() === "Selected Filters") {
    // Scan a window of candidate lines for the first non-blank row matching a rule
    for (let i = 4; i < Math.min(lines.length, 10); i++) {
      const cols = parseCsvLine(lines[i]);
      if (cols.every((c) => c.trim() === "" || c.trim() === '""')) continue; // skip blanks
      for (const rule of DETECTION_RULES) {
        if (rule.check(cols)) {
          return { reportType: rule.type, headerRowIndex: i, dataStartIndex: i + 1, headers: cols };
        }
      }
    }
  }

  return {
    reportType: null, headerRowIndex: -1, dataStartIndex: -1, headers: [],
    error: "This file doesn't match any known source format. Please upload an unmodified CSV export.",
  };
}
```

A tiny dependency-free CSV line parser is used here (not PapaParse) because we only need to tokenize a handful of candidate header rows before deciding the format:

```typescript
/** Minimal CSV line tokenizer that respects quoted fields */
function parseCsvLine(line: string): string[] {
  const result: string[] = [];
  let current = "";
  let inQuotes = false;
  for (let i = 0; i < line.length; i++) {
    const ch = line[i];
    if (ch === '"') inQuotes = !inQuotes;
    else if (ch === "," && !inQuotes) { result.push(current.trim()); current = ""; }
    else current += ch;
  }
  result.push(current.trim());
  return result;
}
```

**Porting notes**

- Order matters: list more-specific rules before broad ones, and use **negative** checks (`!h.includes(...)`) to disambiguate sources that share marker columns.
- The preamble sentinel (`"Selected Filters"`) and the scan window (`lines 4–10`) are tuned to one source family — adjust to your exports. The point is: *detect format, then locate the header, then match a rule*.
- Always return a structured `error` for unrecognized files so the endpoint can give the user an actionable message instead of a 500.

---

## 4. Column Mapping & Coercion

### 4.1 The column-map registry

Each source gets a record mapping **its** CSV header → **your** canonical DB column. Shared fragments (spread with `...`) keep common columns DRY across related sources.

```typescript
// column-maps.ts

const SHARED_CAMPAIGN_COLS: Record<string, string> = {
  CAMPAIGN_ID: "campaignId",
  CAMPAIGN_NAME: "campaignName",
  CAMPAIGN_STATUS: "campaignStatus",
  BRAND_NAME: "brandName",
  // ...
};

const SHARED_METRIC_COLS: Record<string, string> = {
  TOTAL_IMPRESSIONS: "impressions",
  TOTAL_CLICKS: "clicks",
  TOTAL_CTR: "ctr",
  TOTAL_GMV: "gmv",
  TOTAL_CONVERSIONS: "conversions",
  TOTAL_ROI: "roi",
  // ...
};

export const COLUMN_MAPS: Record<ReportType, Record<string, string>> = {
  im_summary: {
    ...SHARED_CAMPAIGN_COLS,
    KEYWORD_COUNT: "keywordCount",
    PRODUCT_COUNT: "productCount",
    ...SHARED_METRIC_COLS,
  },
  im_campaign_daily: {
    METRICS_DATE: "metricsDate",
    ...SHARED_CAMPAIGN_COLS,
    CAMPAIGN_UPTIME_PERCENTAGE: "campaignUptimePercent",
    ...SHARED_METRIC_COLS,
  },
  // ... one entry per source/report
};
```

The same idea, in its **flat single-table** form, normalizes three differently-named sources onto one unified row shape — note how `Revenue` / `gmv` / `Revenue (GMV)` all collapse to `revenue`:

```typescript
// csv-parser.ts — simple variant
const COLUMN_MAPS: Record<string, Record<string, string>> = {
  sourceA: { "Campaign Name": "campaignName", Spend: "spend",        Revenue: "revenue",       Date: "date" },
  sourceB: { campaign_name: "campaignName",   spend: "spend",        gmv: "revenue",           date: "date" },
  sourceC: { "Campaign Name": "campaignName", "Total Spend": "spend", "Revenue (GMV)": "revenue", Date: "date" },
};
```

### 4.2 Type coercion

Two declarative allow-lists drive coercion: `STRING_COLUMNS` (never parse as numbers, even if they look numeric) and `TEXT_COLUMNS` (values that contain `%` and must stay text).

```typescript
// column-maps.ts
export const STRING_COLUMNS = new Set([
  "campaignId", "campaignName", "campaignStatus", "brandName",
  "keyword", "matchType", "searchQuery", "productName", "city",
  "metricsDate", "productId", "ctr", "a2cRate", // ctr/a2cRate hold "12.3%" strings
]);
```

```typescript
// parsers.ts
/** Convert a raw CSV string cell to the appropriate JS type */
function parseValue(value: string, dbCol: string): unknown {
  // Null-like sentinels
  if (value === "" || value === "NA" || value === "na" || value === "-") {
    return STRING_COLUMNS.has(dbCol)
      ? (value === "NA" || value === "na" || value === "-" ? null : "")
      : null;
  }

  // Declared string columns are never coerced to numbers
  if (STRING_COLUMNS.has(dbCol)) return value;

  // Numeric: strip thousands separators and trailing %
  const num = Number(value.replace(/,/g, "").replace(/%$/, ""));
  if (isNaN(num)) return null;
  return num;
}
```

The mapping + coercion loop walks header positions and applies the per-column rule:

```typescript
for (let j = 0; j < headers.length; j++) {
  const csvCol = headers[j];
  const dbCol = columnMap[csvCol];
  if (!dbCol) continue;                                   // ignore unmapped columns
  const rawValue = (rawRow[j] || "").trim().replace(/^"|"$/g, "");
  mapped[dbCol] = parseValue(rawValue, dbCol);
}
```

The **simple variant** coerces inline with plain `parseFloat`/`parseInt` plus computed fields (note derived `cpa`/`roas` guarded against divide-by-zero):

```typescript
spend:        parseFloat(mapped.spend) || 0,
impressions:  parseInt(mapped.impressions) || 0,
conversions:  parseInt(mapped.conversions) || 0,
revenue:      parseFloat(mapped.revenue) || 0,
cpa:          parseInt(mapped.conversions) > 0 ? parseFloat(mapped.spend) / parseInt(mapped.conversions) : null,
platformRoas: parseFloat(mapped.spend) > 0 ? parseFloat(mapped.revenue) / parseFloat(mapped.spend) : null,
```

**Porting notes**

- Keep coercion data-driven (sets/maps), not branch-per-field — it scales to dozens of columns.
- Treat percentage and ID-like fields as strings; numeric coercion silently corrupts `"0012"` or `"12.3%"`.
- Normalize source-specific null sentinels (`"NA"`, `"-"`, `""`) in one place.

---

## 5. Upload Endpoint

The route validates access + input, enforces a size cap, archives the raw file, parses + inserts, and (here) provisions a trial on first upload. Returns the inserted row count and the archive key.

```typescript
// app/api/<tenant>/[tenantId]/upload/route.ts
import { NextRequest, NextResponse } from "next/server";
import { requireBrandAccess } from "@/lib/auth-guard";
import { parseAndInsertCSV } from "@/lib/csv-parser";
import { uploadToS3 } from "@/lib/s3";

export async function POST(req: NextRequest, { params }: { params: Promise<{ brandId: string }> }) {
  const { brandId } = await params;

  // 1. AuthZ — caller must own this tenant
  const access = await requireBrandAccess(brandId);
  if ("error" in access) return access.error;

  // 2. Parse multipart form
  const formData = await req.formData();
  const file = formData.get("file") as File;
  const platformConnectionId = formData.get("platformConnectionId") as string;
  const platform = formData.get("platform") as string;            // optional in auto-detect variant
  if (!file || !platformConnectionId || !platform) {
    return NextResponse.json({ error: "Missing fields" }, { status: 400 });
  }

  // 3. Size guard (reject before reading into memory-bound text())
  const MAX_FILE_SIZE = 50 * 1024 * 1024; // 50MB
  if (file.size > MAX_FILE_SIZE) {
    return NextResponse.json({ error: "File too large. Maximum size is 50MB." }, { status: 413 });
  }

  const csvContent = await file.text();

  // 4. Archive raw original (optional — see §7) BEFORE mutating state, so the
  //    untouched source is always recoverable for replay/debugging
  const s3Key = `uploads/${brandId}/${platform}/${Date.now()}-${file.name}`;
  await uploadToS3(s3Key, csvContent, "text/csv");

  // 5. Parse + batch insert
  const result = await parseAndInsertCSV(csvContent, brandId, platformConnectionId, platform);

  return NextResponse.json({ ...result, rawDataS3Key: s3Key });
}
```

PapaParse handles the heavy parsing. In the **simple variant**, `header: true` yields keyed objects; the column map projects them onto canonical fields:

```typescript
// csv-parser.ts
import Papa from "papaparse";

const parsed = Papa.parse(csvContent, { header: true, skipEmptyLines: true });

const rows = parsed.data.map((row: any) => {
  const mapped: Record<string, any> = {};
  for (const [csvCol, dbCol] of Object.entries(columnMap)) {
    if (row[csvCol] !== undefined) mapped[dbCol] = row[csvCol];
  }
  return { brandId, platformConnectionId, /* ...coerced fields... */ };
});

if (rows.length > 0) await db.insert(dailyMetrics).values(rows);
return { rowsInserted: rows.length };
```

In the **auto-detect variant**, the preamble is sliced off before parsing (`header: false`, since the real header was already located by the detector), and inserts are chunked with per-source dedupe keys:

```typescript
// parsers.ts
const dataContent = lines.slice(detection.dataStartIndex).join("\n");
const parsed = Papa.parse(dataContent, { header: false, skipEmptyLines: true });

const BATCH_SIZE = 500;
for (let i = 0; i < rows.length; i += BATCH_SIZE) {
  const batch = rows.slice(i, i + BATCH_SIZE);
  if (batch.length > 0) {
    await db.insert(table).values(batch as any).onConflictDoNothing(); // dedupe via unique index
  }
}
```

> **Dedupe** relies on a `uniqueIndex` per table matching that source's natural key (see §6). `onConflictDoNothing()` makes re-uploading the same export idempotent — accurate duplicate counting requires those unique indexes to exist.

---

## 6. Database Schema

The reference schema (Drizzle ORM → Postgres) has three layers: a **unified metrics table** (simple variant), **batch/file bookkeeping**, and **per-source typed tables** (auto-detect variant). Below is portable SQL.

```sql
-- Unified table (simple variant): all sources normalized into one shape
CREATE TABLE daily_metrics (
  id                     uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  brand_id               uuid NOT NULL REFERENCES brands(id),
  platform_connection_id uuid NOT NULL REFERENCES platform_connections(id),
  date                   date NOT NULL,
  campaign_name          varchar(500),
  campaign_id_external   varchar(255),
  keyword                varchar(500),
  ad_group               varchar(500),
  spend                  real    DEFAULT 0,
  impressions            integer DEFAULT 0,
  clicks                 integer DEFAULT 0,
  conversions            integer DEFAULT 0,
  revenue                real    DEFAULT 0,
  cpa                    real,
  platform_roas          real,
  raw_data_s3_key        text,             -- pointer back to archived original
  created_at             timestamp NOT NULL DEFAULT now()
);

-- Batch bookkeeping: one row per upload action
CREATE TABLE upload_batches (
  id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  brand_id         uuid NOT NULL REFERENCES brands(id),
  uploaded_by      uuid NOT NULL REFERENCES users(id),
  uploaded_at      timestamp NOT NULL DEFAULT now(),
  file_count       integer DEFAULT 0,
  status           text DEFAULT 'processing',  -- processing | done | failed
  date_range_start date,
  date_range_end   date
);

-- Per-file bookkeeping: detection result + outcome per file in a batch
CREATE TABLE upload_files (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  batch_id      uuid NOT NULL REFERENCES upload_batches(id),
  file_name     text NOT NULL,
  report_type   varchar(50),                  -- detected source/report type (nullable = unrecognized)
  row_count     integer DEFAULT 0,
  s3_key        text,                         -- archived original
  status        text DEFAULT 'detecting',     -- detecting | inserted | failed
  error_message text
);

-- Example per-source typed table (auto-detect variant)
CREATE TABLE im_campaign_daily (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  brand_id        uuid NOT NULL REFERENCES brands(id),
  upload_batch_id uuid NOT NULL REFERENCES upload_batches(id),
  metrics_date    date,
  campaign_id     varchar(255),
  campaign_name   varchar(500),
  campaign_status varchar(100),
  impressions     real DEFAULT 0,
  clicks          real DEFAULT 0,
  ctr             varchar(20),                 -- kept text: holds "12.3%"
  gmv             real DEFAULT 0,
  conversions     real DEFAULT 0,
  roi             real DEFAULT 0,
  created_at      timestamp NOT NULL DEFAULT now()
);

-- Natural-key unique index powers onConflictDoNothing() idempotent re-uploads
CREATE UNIQUE INDEX uq_im_campaign_daily
  ON im_campaign_daily (brand_id, campaign_id, metrics_date);
CREATE INDEX idx_im_campaign_daily_brand
  ON im_campaign_daily (brand_id, upload_batch_id);
```

Per-source dedupe keys (the columns each unique index must cover) are declared alongside the parser, e.g.:

```
im_campaign_daily : (brand_id, campaign_id, metrics_date)
im_granular       : (brand_id, campaign_id, metrics_date, keyword, city, product_name)
im_summary        : (brand_id, campaign_id)
```

---

## 7. Dependencies

### npm packages

| Package | Purpose | Required? |
|---|---|---|
| `papaparse` | CSV parsing (`header: true/false`, `skipEmptyLines`) | **Required** |
| `@types/papaparse` | Types (dev) | Required (TS) |
| `drizzle-orm` | DB access, `insert().values().onConflictDoNothing()` | Required (or swap your ORM/driver) |
| `pg` / Postgres driver | Database connection | **Required** |
| `@aws-sdk/client-s3` | Raw-file archival to S3-compatible storage | **Optional** (archival only) |
| `next` | Route handler / `NextRequest` form parsing | Swappable (any HTTP framework) |

### Environment variables

| Var | Purpose | Required? |
|---|---|---|
| `DATABASE_URL` | Postgres connection string | **Required** |
| `S3_BUCKET` | Archive bucket name | Optional (archival) |
| `AWS_ACCESS_KEY_ID` | Object-storage access key | Optional (archival) |
| `AWS_SECRET_ACCESS_KEY` | Object-storage secret key | Optional (archival) |
| `AWS_S3_ENDPOINT` | Custom endpoint — works with any S3-compatible store (MinIO, Vultr, R2, Backblaze) via `forcePathStyle: true` | Optional (archival) |
| `AWS_REGION` | Region (often a placeholder like `us-east-1` for non-AWS stores) | Optional (archival) |

> **S3 archival is optional.** If you drop it, delete the `uploadToS3` call in the endpoint and the `*_s3_key` columns. The parse + insert path has no dependency on object storage. The reference `s3.ts` uses `forcePathStyle: true` + a custom `AWS_S3_ENDPOINT`, so it works with any S3-compatible provider — not just AWS.

The archival client (drop-in for any S3-compatible store):

```typescript
// s3.ts
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";

export const s3 = new S3Client({
  region: process.env.AWS_REGION!,
  endpoint: process.env.AWS_S3_ENDPOINT!,        // any S3-compatible endpoint
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID!,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY!,
  },
  forcePathStyle: true,
});

export async function uploadToS3(key: string, body: Buffer | string, contentType = "application/octet-stream") {
  await s3.send(new PutObjectCommand({ Bucket: process.env.S3_BUCKET!, Key: key, Body: body, ContentType: contentType }));
  return key;
}
```

---

## 8. Porting Checklist

- [ ] **List your sources.** Enumerate every upstream platform whose CSV you'll ingest and grab a real sample export from each.
- [ ] **Define your canonical schema.** Decide the unified DB column names that all sources map onto.
- [ ] **Build the column-map registry.** One `COLUMN_MAPS[source]` entry per source: `"Their Header" → yourDbColumn`. Use spread fragments for shared columns.
- [ ] **Populate `STRING_COLUMNS` / `TEXT_COLUMNS`.** List every field that must stay text (IDs, percentages, dates-as-strings, status enums).
- [ ] **Write detection rules.** For each source, pick marker columns (presence + absence) that uniquely identify its header. Order specific rules before broad ones.
- [ ] **Handle preamble formats.** If any source prepends metadata rows, set the sentinel + scan window and locate the real header row.
- [ ] **Adjust `parseValue` sentinels.** Map your sources' null markers (`""`, `"NA"`, `"-"`, etc.) and number formatting (thousands separators, `%`, currency symbols).
- [ ] **Create tables + unique indexes.** Define per-source natural-key `uniqueIndex` so `onConflictDoNothing()` makes re-uploads idempotent.
- [ ] **Wire the endpoint.** AuthZ → validate fields → size cap → (archive) → parse + batch insert → return `{ rowsInserted, rawDataS3Key }`.
- [ ] **Tune `BATCH_SIZE`.** Default 500; lower if rows are wide or your driver caps bound parameters.
- [ ] **Decide on archival.** Keep S3 (set the 5 env vars) for replay/audit, or drop it and remove `*_s3_key` columns.
- [ ] **Unit-test the pure stages.** Feed sample headers to `detectReportType` and sample cells to `parseValue` — no DB/network needed.
- [ ] **Test the unrecognized-file path.** Confirm a junk CSV returns a structured, user-actionable error (not a 500).

---

*Built by [Quantana](https://quantana.in). MIT licensed.*
