# Data-Freshness Tracking

> Per-data-source staleness tracking for any ingestion, sync, or scraper SaaS. Each data source gets a freshness status — `fresh`, `stale`, or `outdated` — derived from the age of its most recent successful upload. Surfaces as a compact dashboard widget and a full onboarding checklist so users always know which sources need a refresh.

## 1. Overview

When an app ingests data from multiple external sources (file uploads, API syncs, scrapers, connectors), the data behind each source decays at its own pace. A dashboard that silently serves month-old numbers erodes trust.

This feature computes a **freshness status per data source** from the timestamp of its latest successful ingest, classifies it against configurable age thresholds, and renders two views:

- **Freshness widget** — a small dashboard card showing an overall "X of Y sources fresh" progress bar plus a callout for the stalest source.
- **Onboarding checklist** — a full upload/sync page listing every source (required vs. optional), its last-ingested time, row count, and a status badge.

A single status endpoint backs both UIs. The model is generic: anything with a "source identifier" and a "last successful ingest timestamp" plugs in.

## 2. Flow

```
                 ┌──────────────────────────────────────────────┐
                 │  upload_files / upload_batches  (per <Org>)   │
                 │  one row per ingest, tagged with sourceType   │
                 └───────────────────────┬──────────────────────┘
                                         │
                       GROUP BY sourceType, MAX(ingestedAt)
                                         │
                                         ▼
                 ┌──────────────────────────────────────────────┐
                 │   latest ingest timestamp per data source     │
                 └───────────────────────┬──────────────────────┘
                                         │
                        daysSince = now - lastIngestedAt
                                         │
            ┌────────────────────────────┼────────────────────────────┐
            ▼                            ▼                             ▼
     daysSince < 7              7 ≤ daysSince < 30              daysSince ≥ 30
            │                            │                             │
         "fresh"                      "stale"                     "outdated"
            └────────────────────────────┼────────────────────────────┘
                                         │   (no rows → "never")
                                         ▼
                 ┌──────────────────────────────────────────────┐
                 │  GET /api/<org>/{id}/source-status  → JSON    │
                 │  { sources[], freshCount, totalCount }        │
                 └───────────────┬───────────────┬──────────────┘
                                 │               │
                                 ▼               ▼
                      ┌──────────────────┐  ┌──────────────────────┐
                      │ Freshness widget │  │ Onboarding checklist │
                      │ (dashboard card) │  │ (upload/sync page)   │
                      └──────────────────┘  └──────────────────────┘
```

## 3. Freshness Model

A four-state enum plus a pure classifier function. Age thresholds are plain constants — override them per source if you need custom decay rates (see Porting Checklist).

```ts
// lib/source-types.ts
export const FRESH_THRESHOLD_DAYS = 7;
export const STALE_THRESHOLD_DAYS = 30;

export type FreshnessStatus = "never" | "fresh" | "stale" | "outdated";

export interface SourceTypeMeta {
  type: string;          // stable source identifier, e.g. "stripe_payouts"
  name: string;          // human label
  description: string;
  required: boolean;     // required = blocks onboarding completion
}

// Describe every data source your app ingests.
export const SOURCE_TYPE_META: SourceTypeMeta[] = [
  { type: "summary",      name: "Campaign Summary", description: "Aggregate metrics (ROI, revenue, spend)", required: true },
  { type: "search_query", name: "Search Query",     description: "Query-level performance & conversions",   required: true },
  { type: "granular",     name: "Granular Breakdown", description: "Performance by region and product",     required: true },
  { type: "daily",        name: "Daily Metrics",     description: "Daily time-series and uptime",            required: false },
  { type: "placement",    name: "Placement",         description: "Placement and keyword performance",       required: false },
  // ...one entry per source
];

export function computeFreshnessStatus(lastIngestedAt: Date | null): FreshnessStatus {
  if (!lastIngestedAt) return "never";
  const daysSince = Math.floor((Date.now() - lastIngestedAt.getTime()) / (1000 * 60 * 60 * 24));
  if (daysSince < FRESH_THRESHOLD_DAYS) return "fresh";
  if (daysSince < STALE_THRESHOLD_DAYS) return "stale";
  return "outdated";
}
```

| Status     | Condition                          | Meaning                                |
|------------|------------------------------------|----------------------------------------|
| `never`    | no successful ingest on record     | source not yet connected               |
| `fresh`    | `daysSince < 7`                    | up to date                             |
| `stale`    | `7 ≤ daysSince < 30`               | aging — refresh soon                   |
| `outdated` | `daysSince ≥ 30`                   | numbers no longer trustworthy          |

## 4. Status Endpoint

`GET /api/<org>/{id}/source-status` resolves the latest ingest per source via a single grouped query (`MAX(ingestedAt)` joined batches → files), classifies each, and returns counts. `null` timestamps fall through to `"never"`.

```ts
// app/api/<org>/[orgId]/source-status/route.ts
import { NextResponse } from "next/server";
import { requireOrgAccess } from "@/lib/auth-guard";
import { db } from "@/lib/db";
import { uploadFiles, uploadBatches } from "@/lib/db/schema";
import { eq, sql, and } from "drizzle-orm";
import { SOURCE_TYPE_META, computeFreshnessStatus } from "@/lib/source-types";

export async function GET(
  _req: Request,
  { params }: { params: Promise<{ orgId: string }> }
) {
  const { orgId } = await params;
  const access = await requireOrgAccess(orgId);
  if ("error" in access) return access.error;

  // Latest successful ingest per source type (one grouped query).
  const latestUploads = await db
    .select({
      sourceType: uploadFiles.sourceType,
      lastIngestedAt: sql<string>`MAX(${uploadBatches.uploadedAt})`.as("last_ingested_at"),
      rowCount: sql<number>`SUM(${uploadFiles.rowCount})`.as("row_count"),
    })
    .from(uploadFiles)
    .innerJoin(uploadBatches, eq(uploadFiles.batchId, uploadBatches.id))
    .where(
      and(
        eq(uploadBatches.orgId, orgId),
        eq(uploadFiles.status, "complete")
      )
    )
    .groupBy(uploadFiles.sourceType);

  const uploadMap = new Map(latestUploads.map((u) => [u.sourceType, u]));

  let freshCount = 0;
  const sources = SOURCE_TYPE_META.map((meta) => {
    const upload = uploadMap.get(meta.type);
    const lastIngestedAt = upload?.lastIngestedAt ? new Date(upload.lastIngestedAt) : null;
    const status = computeFreshnessStatus(lastIngestedAt);
    if (status === "fresh") freshCount++;

    return {
      type: meta.type,
      name: meta.name,
      description: meta.description,
      required: meta.required,
      lastIngestedAt: lastIngestedAt?.toISOString() ?? null,
      rowCount: upload?.rowCount ?? null,
      status,
    };
  });

  return NextResponse.json({
    sources,
    freshCount,
    totalCount: SOURCE_TYPE_META.length,
  });
}
```

**Response shape:**

```jsonc
{
  "sources": [
    { "type": "summary", "name": "Campaign Summary", "description": "...",
      "required": true, "lastIngestedAt": "2026-06-18T09:00:00.000Z",
      "rowCount": 1240, "status": "fresh" }
  ],
  "freshCount": 3,
  "totalCount": 5
}
```

## 5. UI Components

Both components fetch the same endpoint on mount and render off its JSON. No prop drilling of data — pass an `orgId` and they self-fetch.

### Freshness widget (dashboard card)

A compact card: overall progress bar (`freshCount / totalCount`) plus a single callout for the stalest non-fresh source and its age in days.

```tsx
// components/source-freshness-widget.tsx  (summary)
function statusColor(status: FreshnessStatus) {
  switch (status) {
    case "fresh":    return "text-positive";
    case "stale":    return "text-warning";
    case "outdated": return "text-destructive";
    default:         return "text-muted";
  }
}

export function SourceFreshnessWidget({ orgId, orgSlug }: { orgId: string; orgSlug: string }) {
  const [data, setData] = useState<SourceStatusResponse | null>(null);

  useEffect(() => {
    fetch(`/api/<org>/${orgId}/source-status`)
      .then((r) => r.json())
      .then(setData)
      .catch(() => {});
  }, [orgId]);

  if (!data) return null;
  const { freshCount, totalCount, sources } = data;
  const allFresh = freshCount === totalCount;
  const progressPercent = Math.round((freshCount / totalCount) * 100);

  // Stalest = oldest source that is neither "never" nor "fresh".
  const stalest = sources
    .filter((s) => s.status !== "never" && s.status !== "fresh")
    .sort((a, b) => new Date(a.lastIngestedAt!).getTime() - new Date(b.lastIngestedAt!).getTime())[0];

  return (
    <div className="rounded-xl border bg-white p-4 shadow">
      <div className="flex items-center justify-between mb-2">
        <h3 className="text-sm font-semibold">Data Freshness</h3>
        <Link href={`/<org>/${orgSlug}/upload`} className="text-xs hover:underline">
          Manage sources →
        </Link>
      </div>
      <div className="flex items-center justify-between text-xs mb-1">
        <span className={allFresh ? "text-positive font-medium" : "text-muted"}>
          {allFresh ? "All sources up to date" : `${freshCount} of ${totalCount} sources fresh`}
        </span>
      </div>
      <div className="h-1.5 w-full rounded-full bg-gray-100">
        <div className={`h-1.5 rounded-full ${allFresh ? "bg-positive" : "bg-blue-500"}`}
             style={{ width: `${progressPercent}%` }} />
      </div>
      {stalest && (
        <p className={`mt-2 text-xs ${statusColor(stalest.status)}`}>
          {stalest.name} is {Math.floor((Date.now() - new Date(stalest.lastIngestedAt!).getTime()) / 86_400_000)} days old
        </p>
      )}
    </div>
  );
}
```

### Onboarding checklist (upload/sync page)

Full list split into **Required** (blocks onboarding) and **Optional** (deeper insights). Each row shows name, description, a relative "time ago", row count, and a colored status badge. Exposes a `refetch` callback so the page can refresh it after a new upload completes.

```tsx
// components/source-status-checklist.tsx  (summary)
function StatusBadge({ status }: { status: FreshnessStatus }) {
  const config = {
    never:    { label: "Never connected", bg: "bg-gray-100 text-gray-600" },
    fresh:    { label: "Fresh",           bg: "bg-green-50 text-green-700" },
    stale:    { label: "Stale",           bg: "bg-amber-50 text-amber-700" },
    outdated: { label: "Outdated",        bg: "bg-red-50 text-red-700" },
  };
  const { label, bg } = config[status];
  return <span className={`inline-flex rounded-full px-2 py-0.5 text-xs font-medium ${bg}`}>{label}</span>;
}

function timeAgo(dateStr: string): string {
  const days = Math.floor((Date.now() - new Date(dateStr).getTime()) / 86_400_000);
  if (days === 0) return "Today";
  if (days === 1) return "Yesterday";
  if (days < 30) return `${days} days ago`;
  const months = Math.floor(days / 30);
  return `${months} month${months > 1 ? "s" : ""} ago`;
}

export function SourceStatusChecklist({ orgId, onRefetchReady }: {
  orgId: string; onRefetchReady?: (refetch: () => void) => void;
}) {
  const [data, setData] = useState<SourceStatusResponse | null>(null);
  const [loading, setLoading] = useState(true);

  const fetchStatus = () => {
    setLoading(true);
    fetch(`/api/<org>/${orgId}/source-status`)
      .then((r) => r.json())
      .then((d) => { setData(d); setLoading(false); })
      .catch(() => setLoading(false));
  };

  useEffect(() => { fetchStatus(); }, [orgId]);
  useEffect(() => { onRefetchReady?.(fetchStatus); }, [orgId]);

  if (loading && !data) return <div className="animate-pulse h-48 rounded-xl bg-gray-100" />;
  if (!data) return null;

  const required = data.sources.filter((s) => s.required);
  const optional = data.sources.filter((s) => !s.required);
  // ...render header ("X of Y sources up to date"), then Required and Optional rows,
  //    each row = <name/description> + timeAgo + rowCount + <StatusBadge />
}
```

## 6. Database Schema

Two tables. A **batch** is one ingest run; **files** are the individual datasets within it, each tagged with a `sourceType`. The status endpoint reads only these — your domain tables are irrelevant to freshness.

```ts
// lib/db/schema.ts (Drizzle / Postgres)
export const uploadBatches = pgTable("upload_batches", {
  id: uuid("id").primaryKey().defaultRandom(),
  orgId: uuid("org_id").notNull().references(() => orgs.id),
  uploadedBy: uuid("uploaded_by").notNull().references(() => users.id),
  uploadedAt: timestamp("uploaded_at").defaultNow().notNull(), // <- the freshness clock
  fileCount: integer("file_count").default(0),
  status: uploadStatusEnum("status").default("processing"),
  dateRangeStart: date("date_range_start"),
  dateRangeEnd: date("date_range_end"),
});

export const uploadFiles = pgTable("upload_files", {
  id: uuid("id").primaryKey().defaultRandom(),
  batchId: uuid("batch_id").notNull().references(() => uploadBatches.id),
  fileName: text("file_name").notNull(),
  sourceType: varchar("source_type", { length: 50 }), // matches SOURCE_TYPE_META.type
  rowCount: integer("row_count").default(0),
  s3Key: text("s3_key"),
  status: uploadFileStatusEnum("status").default("detecting"), // "complete" counts toward freshness
  errorMessage: text("error_message"),
});
```

Key points:
- **`uploadBatches.uploadedAt`** is the timestamp the freshness query maxes over — set it when an ingest succeeds.
- **`uploadFiles.sourceType`** ties each ingested dataset back to a `SOURCE_TYPE_META` entry. This is the `GROUP BY` key.
- **`uploadFiles.status = "complete"`** is the filter so in-flight or failed ingests never count as fresh.
- Recommended index: `(orgId, sourceType)` on the join path, or a covering index on `uploadBatches(orgId, uploadedAt)`.

## 7. Dependencies

**npm**

```jsonc
{
  "dependencies": {
    "next": "^14 || ^15",            // App Router route handlers + Link
    "react": "^18 || ^19",
    "drizzle-orm": "^0.30+",         // grouped MAX/SUM query (or your ORM/raw SQL)
    "pg": "^8"                        // Postgres driver
  }
}
```

**Environment variables**

| Var            | Purpose                                                   |
|----------------|----------------------------------------------------------|
| `DATABASE_URL` | Postgres connection string for the uploads tables        |

No external API keys required — freshness is computed entirely from your own ingest timestamps. UI is plain React + Tailwind class names (swap for your design tokens).

## 8. Porting Checklist

- [ ] Copy `lib/source-types.ts` — fill `SOURCE_TYPE_META` with your real data sources (`type`, `name`, `description`, `required`).
- [ ] Tune `FRESH_THRESHOLD_DAYS` / `STALE_THRESHOLD_DAYS` for your data's decay rate.
- [ ] **Configurable thresholds per source:** for mixed cadences (hourly API sync vs. monthly export), extend `SourceTypeMeta` with optional `freshDays` / `staleDays` and have `computeFreshnessStatus(lastIngestedAt, meta)` prefer those over the global constants. Sources without overrides fall back to the defaults.
- [ ] Add/rename the `uploadBatches` + `uploadFiles` tables; ensure `uploadedAt` is set on successful ingest and `sourceType` is populated.
- [ ] Add the `source-status` route handler; wire `requireOrgAccess` to your auth/tenancy guard.
- [ ] Confirm the grouped query filters on `status = "complete"` (or your terminal-success enum value).
- [ ] Drop in `SourceFreshnessWidget` on the dashboard and `SourceStatusChecklist` on the upload/sync page; map color classes to your tokens.
- [ ] Pass `onRefetchReady` from the upload page and call the returned `refetch` after an ingest completes so the checklist updates live.
- [ ] Add a DB index on the `(orgId, sourceType)` join path for large upload histories.
- [ ] (Optional) Emit a notification/email when any required source crosses into `outdated`.

---

*Built by [Quantana](https://quantana.in). MIT licensed.*
