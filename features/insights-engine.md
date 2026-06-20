# Pluggable Multi-Pass Insights Engine

> A domain-agnostic analytics framework that runs N independent "passes" over your data, each emitting structured insight objects, then feeds the collected results to an LLM that writes an executive summary. Every pass shares one contract, so you add, remove, or reorder analysis modules without touching the orchestrator. The example below is ad-tech (campaigns, ROAS, wasted spend) — but the framework is the product, and the domain passes are swappable.

---

## 1. Overview

The Insights Engine turns a pile of rows into a ranked, human-readable list of findings plus an AI executive summary — on every new data upload.

The core idea is **pluggable passes**. A "pass" is a self-contained function that:

1. Reads some slice of the data (one SQL aggregate, one grouping, one join).
2. Applies domain thresholds (e.g. "ROI below 2x is bad", "zero conversions = waste").
3. Emits zero or more **insight objects** in a shared shape (`category`, `severity`, `title`, `summary`, `detail` payload).

The orchestrator runs the passes in sequence, concatenates everything they emit into one flat array, hands that array to a final **executive-summary pass** (which calls an LLM), and persists the whole batch to one table.

**Why this is powerful:**

- **Separation of concerns.** Each pass is ~70 lines and owns exactly one question. You can read, test, and reason about it in isolation.
- **Composability.** Adding analysis = adding a file + one line in the orchestrator. No shared mutable state, no cross-pass coupling (except the summary pass, which deliberately reads the others' outputs).
- **Uniform storage and rendering.** Because every pass emits the same object shape into the same table, your UI renders all insights generically — a card per row, colored by `severity`, expandable into `detail`.
- **Domain-portable.** The framework knows nothing about ads. Swap the passes and you have a churn engine, a security-findings engine, a financial-anomaly engine. **You keep the orchestrator, the contract, the table, and the summary pass; you rewrite only the domain logic inside each pass.**

A porting reader should mentally substitute their own nouns: replace *campaigns* with *users*, *ROAS* with *retention*, *wasted spend* with *failed jobs*. The skeleton does not change.

---

## 2. Architecture

```
                        ┌──────────────────────────────────────────────┐
                        │          generateInsights(scopeId,           │
                        │                  uploadBatchId)              │
                        │              ── the orchestrator ──          │
                        └──────────────────────────────────────────────┘
                                            │
          ┌─────────────────────────────────┼─────────────────────────────────┐
          ▼                                 ▼                                 ▼
   ┌─────────────┐                  ┌─────────────┐                   ┌─────────────┐
   │   Data /    │                  │   Data /    │                   │   Data /    │
   │  Database   │                  │  Database   │       ...         │  Database   │
   └──────┬──────┘                  └──────┬──────┘                   └──────┬──────┘
          ▼                                ▼                                 ▼
   ┌─────────────┐   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
   │  Pass 1     │   │  Pass 2     │  │  Pass 3     │  │   ...       │  │  Pass N     │
   │  top_line   │   │ campaign_roi│  │   waste     │  │             │  │ keyword_sov │
   │ (SQL agg →  │   │ (rank +     │  │ (group +    │  │             │  │ (group +    │
   │  threshold) │   │  threshold) │  │  LLM tag)   │  │             │  │  threshold) │
   └──────┬──────┘   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
          │                 │                │                │                │
          └─────────────────┴────────────────┴────────────────┴────────────────┘
                                            │
                                            ▼
                            ┌──────────────────────────────┐
                            │   allInsights: InsightRow[]   │   ← flat array, one shape
                            └───────────────┬──────────────┘
                                            │  (passed in)
                                            ▼
                            ┌──────────────────────────────┐
                            │   Pass N+1: executiveSummary  │
                            │   build metrics context  ──►  │
                            │   LLM (Claude) ──► {summary,  │
                            │   actions[], health}          │
                            └───────────────┬──────────────┘
                                            │  appended
                                            ▼
                            ┌──────────────────────────────┐
                            │   db.insert(insights)         │   ← single batch write
                            │   (severity, category, title, │
                            │    summary, detail JSONB)     │
                            └──────────────────────────────┘
```

The flow is strictly: **data → pass 1..N → collect insights → LLM summary → store**. Passes 1..N are independent of each other; the summary pass is the only one that consumes prior outputs.

---

## 3. The Pass Interface

Every pass is an `async` function with the **same signature and the same return type**. That single shared contract is what makes the engine pluggable.

```ts
// The contract every domain pass implements.
// scopeId  = whatever you partition data by (here: brandId; in your app: tenantId, accountId, userId…)
// batchId  = the specific upload/run being analyzed
type Pass = (
  scopeId: string,
  batchId: string,
) => Promise<InsightRow[]>;
```

`InsightRow` is the insert shape of the `insights` table. In this codebase it is derived directly from the Drizzle schema, so the type can never drift from the DB:

```ts
// typeof insights.$inferInsert  — the canonical insight object
{
  brandId:       string;     // the scope id  → rename to your tenant key
  uploadBatchId: string;     // the run id
  category:      InsightCategory;   // enum: which pass produced this
  severity:      "critical" | "warning" | "info" | "positive";
  title:         string;     // one-line headline, already formatted with numbers
  summary:       string | null;     // 1–2 sentence human explanation
  detail:        unknown;    // JSONB payload — arbitrary structured drill-down data
  metricValue:   number | null;     // the headline number (for sorting / sparkbars)
  metricUnit:    string | null;     // "x", "₹", "campaigns", "%" …
}
```

**Rules of the contract:**

- A pass returns an **array** so it can emit 0, 1, or many insights. Most emit exactly one; a pass that finds nothing returns `[]` (and the orchestrator simply skips it).
- A pass **never writes to the DB.** It only reads and returns. All writes happen once, in the orchestrator. This keeps passes pure, testable, and reorderable.
- `category` ties the row back to the pass that made it (it's a DB enum — see §7). `severity` drives UI color and the summary pass's prioritization.
- `detail` (JSONB) is the escape hatch: stuff any drill-down structure your UI needs (per-item breakdowns, suggested actions, sub-rankings). It is schemaless on purpose.

To port: keep this contract **verbatim**. Only the enum members in `category` change.

---

## 4. Example Passes

Three real passes, showing the recurring pattern: **read a slice → compute → apply thresholds → return insight object(s).** Substitute your own SQL and thresholds; the shape stays identical.

### 4a. `top_line` — a single SQL aggregate → headline KPI

The simplest pass: one `SUM` query over the dataset, derive ratios, pick a severity from a threshold, emit one insight.

```ts
export async function runTopLinePass(brandId, uploadBatchId): Promise<InsightRow[]> {
  const rows = await db.select({
      totalSpend:       sql<number>`coalesce(sum(${imSummary.budgetBurnt}), 0)`,
      totalGmv:         sql<number>`coalesce(sum(${imSummary.gmv}), 0)`,
      totalConversions: sql<number>`coalesce(sum(${imSummary.conversions}), 0)`,
      totalClicks:      sql<number>`coalesce(sum(${imSummary.clicks}), 0)`,
      totalImpressions: sql<number>`coalesce(sum(${imSummary.impressions}), 0)`,
      campaignCount:    sql<number>`count(*)`,
    })
    .from(imSummary)
    .where(and(eq(imSummary.brandId, brandId), eq(imSummary.uploadBatchId, uploadBatchId)));

  const data = rows[0];
  if (!data || data.totalSpend === 0) return [];   // nothing to say → emit nothing

  const blendedRoi = data.totalGmv / data.totalSpend;
  // ... derive ctr, efficiencyScore via weighted normalized sub-scores ...

  const severity = blendedRoi < 5 ? "critical" : blendedRoi < 10 ? "warning" : "positive";

  return [{
    brandId, uploadBatchId,
    category: "top_line",
    severity,
    title: `Blended ROI: ${blendedRoi.toFixed(1)}x`,
    summary: `₹${Math.round(data.totalSpend).toLocaleString("en-IN")} spent across ${data.campaignCount} campaigns generating ₹${Math.round(data.totalGmv).toLocaleString("en-IN")} GMV`,
    detail: { totalSpend: data.totalSpend, totalGmv: data.totalGmv, blendedRoi, efficiencyScore, /* … */ },
    metricValue: blendedRoi,
    metricUnit: "x",
  }];
}
```

> **Pattern:** aggregate → ratio → threshold ladder → one insight. Your churn-engine equivalent: `SUM(active_days)/SUM(signups)` → retention rate → `< 20% critical`.

### 4b. `campaign_roi` — rank items, split into "kill" and "scale" lists

Pulls per-item rows, sorts, partitions by threshold, and packs the full ranking plus actionable sub-lists into `detail`.

```ts
export async function runCampaignRoiPass(brandId, uploadBatchId): Promise<InsightRow[]> {
  const campaigns = await db.select({ /* name, status, spend, roi, … */ })
    .from(imSummary)
    .where(and(eq(imSummary.brandId, brandId), eq(imSummary.uploadBatchId, uploadBatchId)));

  if (campaigns.length === 0) return [];

  const sorted    = [...campaigns].sort((a, b) => (b.roi ?? 0) - (a.roi ?? 0));
  const killList  = sorted.filter((c) => (c.roi ?? 0) < 2  && (c.spend ?? 0) > 0);  // underperformers
  const scaleList = sorted.filter((c) => (c.roi ?? 0) > 15 && (c.spend ?? 0) > 0);  // winners

  return [{
    brandId, uploadBatchId,
    category: "campaign_roi",
    severity: killList.length > 0 ? "warning" : "positive",
    title: `${campaigns.length} campaigns analyzed — ${killList.length} underperforming`,
    summary: killList.length > 0
      ? `${killList.length} campaigns with ROI below 2x are burning ₹${Math.round(killList.reduce((s, c) => s + (c.spend ?? 0), 0)).toLocaleString("en-IN")}`
      : "All campaigns performing above 2x ROI threshold",
    detail: {
      allCampaigns: sorted.map(/* full ranking for the UI table */),
      killList:  killList.map((c) => ({ campaign: c.campaignName, spend: c.spend, roi: c.roi })),
      scaleList: scaleList.map((c) => ({ campaign: c.campaignName, spend: c.spend, roi: c.roi, budget: c.budget })),
      budgetConcentration: { /* top3Pct, bottom3Pct */ },
    },
    metricValue: killList.length,
    metricUnit: "campaigns",
  }];
}
```

> **Pattern:** per-row fetch → sort → threshold-partition into action buckets → embed buckets in `detail`. Your security equivalent: rank findings by CVSS, split into "block now" vs "monitor".

### 4c. `waste` — group + threshold, then enrich with an LLM mid-pass

A pass can call an LLM *itself* (not just the final summary pass). Here, a `GROUP BY … HAVING` finds zero-result spend, then Claude tags each item into categories. The LLM is best-effort — if it fails, the pass degrades gracefully and still emits the insight.

```ts
export async function runWastePass(brandId, uploadBatchId): Promise<InsightRow[]> {
  const wasteRows = await db.select({
      searchQuery:  imSearchQuery.searchQuery,
      totalSpend:   sql<number>`coalesce(sum(${imSearchQuery.budgetBurnt}), 0)`,
      totalConversions: sql<number>`coalesce(sum(${imSearchQuery.conversions}), 0)`,
    })
    .from(imSearchQuery)
    .where(and(eq(imSearchQuery.brandId, brandId), eq(imSearchQuery.uploadBatchId, uploadBatchId)))
    .groupBy(imSearchQuery.searchQuery)
    .having(sql`sum(${imSearchQuery.conversions}) = 0 AND sum(${imSearchQuery.budgetBurnt}) > 0`); // ← the waste condition

  if (wasteRows.length === 0) return [];

  const totalWaste = wasteRows.reduce((s, r) => s + r.totalSpend, 0);
  const topWaste   = wasteRows.sort((a, b) => b.totalSpend - a.totalSpend).slice(0, 50);

  // Optional LLM enrichment — tag each wasted query. Best-effort; failures are swallowed.
  let categorized: Record<string, string> = {};
  try {
    const response = await anthropic.messages.create({
      model: "claude-sonnet-4-6", max_tokens: 1024,
      messages: [{ role: "user", content: `Categorize each query as irrelevant_product / too_broad / competitor_brand / other. Respond with ONLY a JSON object. Queries: ${topWaste.map(w => w.searchQuery).join(", ")}` }],
    });
    const text = response.content[0].type === "text" ? response.content[0].text : "";
    const jsonMatch = text.match(/\{[\s\S]*\}/);
    if (jsonMatch) categorized = JSON.parse(jsonMatch[0]);
  } catch { /* leave uncategorized */ }

  const severity = totalWaste > 20000 ? "critical" : totalWaste > 5000 ? "warning" : "info";

  return [{
    brandId, uploadBatchId,
    category: "waste",
    severity,
    title: `₹${Math.round(totalWaste).toLocaleString("en-IN")} wasted on zero-conversion queries`,
    summary: `${wasteRows.length} search queries triggered ads but generated zero sales`,
    detail: { totalWaste, totalWasteQueries: wasteRows.length, wasteQueries: /* tagged list */, suggestedNegatives: /* derived from tags */, wasteByCategory: /* spend per tag */ },
    metricValue: totalWaste,
    metricUnit: "₹",
  }];
}
```

> **Pattern:** `GROUP BY … HAVING <bad condition>` → optional LLM tagging → threshold severity. The `try/catch` around the LLM is important: enrichment must never break the deterministic insight.

The full engine ships eight passes (`top_line`, `campaign_roi`, `waste`, `budget_overspend`, `cpc_vs_cpm`, `city_performance`, `keyword_sov`, `executive_summary`), all following the same three sub-patterns above.

---

## 5. The Orchestrator

`engine.ts` is the entire framework. It is deliberately dumb: clear old insights, run each pass, concat results, run the summary pass last (handing it the accumulated array), then one batch insert. Adding a pass = one import + one block.

```ts
import { db } from "../db";
import { insights } from "../db/schema";
import { eq } from "drizzle-orm";
import { runTopLinePass } from "./passes/top-line";
import { runCampaignRoiPass } from "./passes/campaign-roi";
import { runWastePass } from "./passes/waste";
import { runBudgetOverspendPass } from "./passes/budget-overspend";
import { runCpcVsCpmPass } from "./passes/cpc-vs-cpm";
import { runCityPerformancePass } from "./passes/city-performance";
import { runKeywordSovPass } from "./passes/keyword-sov";
import { runExecutiveSummaryPass } from "./passes/executive-summary";

export async function generateInsights(
  brandId: string,
  uploadBatchId: string,
): Promise<{ insightCount: number }> {
  // Latest upload replaces all prior insights for this scope.
  await db.delete(insights).where(eq(insights.brandId, brandId));

  const allInsights: typeof insights.$inferInsert[] = [];

  // Passes 1..N — independent, order-agnostic. Each appends what it finds.
  allInsights.push(...await runTopLinePass(brandId, uploadBatchId));          // 1
  allInsights.push(...await runCampaignRoiPass(brandId, uploadBatchId));      // 2
  allInsights.push(...await runWastePass(brandId, uploadBatchId));           // 3
  allInsights.push(...await runBudgetOverspendPass(brandId, uploadBatchId)); // 4
  allInsights.push(...await runCpcVsCpmPass(brandId, uploadBatchId));        // 5
  allInsights.push(...await runCityPerformancePass(brandId, uploadBatchId)); // 6
  allInsights.push(...await runKeywordSovPass(brandId, uploadBatchId));      // 7

  // Pass N+1 — the summary. It RECEIVES the accumulated array.
  allInsights.push(...await runExecutiveSummaryPass(brandId, uploadBatchId, allInsights)); // 8

  // Single batch write.
  if (allInsights.length > 0) {
    await db.insert(insights).values(allInsights);
  }

  return { insightCount: allInsights.length };
}
```

Notes for porting:

- **`delete` then `insert`** = "latest run replaces all." If you want history instead, drop the delete and key reads by `uploadBatchId`.
- Passes run **sequentially** here for simplicity and to keep DB load predictable. Because passes 1..N are independent, you may parallelize them with `Promise.all` — only the summary pass must run last.
- The orchestrator is the **only** place that touches the DB for writes. Passes stay pure.

### Invocation (API route)

The engine is triggered by a thin authenticated `POST` route. Auth + input validation live here; all logic lives in the engine.

```ts
export async function POST(req, { params }: { params: Promise<{ brandId: string }> }) {
  const { brandId } = await params;
  const access = await requireBrandAccess(brandId);   // tenant authz guard
  if ("error" in access) return access.error;

  const { uploadBatchId } = await req.json();
  if (!uploadBatchId) {
    return NextResponse.json({ error: "uploadBatchId required" }, { status: 400 });
  }

  const result = await generateInsights(brandId, uploadBatchId);
  return NextResponse.json(result);   // { insightCount }
}
```

---

## 6. LLM Executive Summary

The last pass is special: instead of querying the DB, it consumes the **outputs of the previous passes** (passed in as `allInsights`), flattens their key metrics into a compact text context, and asks Claude for a summary + ranked actions + a health verdict.

```ts
export async function runExecutiveSummaryPass(
  brandId, uploadBatchId,
  allInsights: typeof insights.$inferInsert[],   // ← receives prior passes' results
): Promise<InsightRow[]> {
  // Pluck the insights produced by earlier passes, by category.
  const topLine     = allInsights.find((i) => i.category === "top_line");
  const campaignRoi = allInsights.find((i) => i.category === "campaign_roi");
  const waste       = allInsights.find((i) => i.category === "waste");
  // … budget, cpcCpm, city …

  // Build a tight, numbers-first context string from their `detail` payloads.
  const metricsContext = `
Total Spend: ₹${topLineData?.totalSpend ?? "N/A"}
Blended ROI: ${topLineData?.blendedRoi ?? "N/A"}x
Efficiency Score: ${topLineData?.efficiencyScore ?? "N/A"}/100
Waste: ₹${wasteData?.totalWaste ?? 0} on ${wasteData?.totalWasteQueries ?? 0} zero-conversion queries
Kill List: ${killList?.length ?? 0} campaigns below 2x ROI
`.trim();

  try {
    const response = await anthropic.messages.create({
      model: "claude-sonnet-4-6",
      max_tokens: 512,
      messages: [{
        role: "user",
        content: `You are an expert analyst. Given these metrics, write:
1. A 3-4 sentence executive summary
2. The top 3 recommended actions ranked by potential impact
3. An overall health assessment: "healthy", "needs_attention", or "critical"
Respond as JSON: { "summary": "...", "actions": ["...","...","..."], "health": "..." }
Metrics:\n${metricsContext}`,
      }],
    });

    const text = response.content[0].type === "text" ? response.content[0].text : "";
    const jsonMatch = text.match(/\{[\s\S]*\}/);
    const parsed = jsonMatch ? JSON.parse(jsonMatch[0])
                             : { summary: "Unable to generate summary", actions: [], health: "needs_attention" };

    return [{
      brandId, uploadBatchId,
      category: "executive_summary",
      severity: parsed.health === "critical" ? "critical"
              : parsed.health === "healthy"  ? "positive"
              : "warning",
      title: `Health: ${parsed.health.replace(/_/g, " ")}`,
      summary: parsed.summary,
      detail: { summary: parsed.summary, actions: parsed.actions, health: parsed.health },
      metricValue: null, metricUnit: null,
    }];
  } catch (err) {
    // Graceful fallback — a failed LLM call must not fail the whole run.
    return [{
      brandId, uploadBatchId,
      category: "executive_summary", severity: "info",
      title: "Executive summary unavailable",
      summary: "Could not generate AI summary. Review individual insight pages for details.",
      detail: { error: String(err) },
      metricValue: null, metricUnit: null,
    }];
  }
}
```

Design points to carry over:

- **The summary is itself an insight.** It's the same `InsightRow` shape with `category: "executive_summary"`, so it stores and renders through the same path as everything else.
- **Map the LLM's verdict onto your severity enum** so the UI colors the summary card consistently (`healthy → positive`, `critical → critical`, else `warning`).
- **Always wrap the LLM in try/catch and ship a deterministic fallback insight.** The engine must succeed even when the model call doesn't.
- **Extract JSON defensively** (`text.match(/\{[\s\S]*\}/)`) rather than trusting the model to return bare JSON.

---

## 7. Database Schema

Everything lands in one table. Generic columns + a JSONB `detail` give you uniform storage with per-pass flexibility.

```sql
-- Two enums drive UI color and "which pass made this".
CREATE TYPE insight_category AS ENUM (
  'top_line', 'campaign_roi', 'waste', 'budget_overspend',
  'cpc_vs_cpm', 'city_performance', 'keyword_sov', 'executive_summary'
  -- ↑ swap these for your domain's pass names
);
CREATE TYPE insight_severity AS ENUM ('critical', 'warning', 'info', 'positive');

CREATE TABLE insights (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  brand_id        UUID NOT NULL REFERENCES brands(id),          -- the scope key (rename: tenant_id…)
  upload_batch_id UUID NOT NULL REFERENCES upload_batches(id),  -- the run id
  category        insight_category NOT NULL,                    -- which pass produced this
  severity        insight_severity NOT NULL,                    -- drives UI color + prioritization
  title           TEXT NOT NULL,                                -- formatted headline
  summary         TEXT,                                         -- 1–2 sentence explanation
  detail          JSONB,                                        -- arbitrary drill-down payload
  metric_value    REAL,                                         -- headline number (sorting/sparkbars)
  metric_unit     VARCHAR(20),                                  -- "x", "₹", "campaigns", "%"
  created_at      TIMESTAMP NOT NULL DEFAULT now()
);

CREATE INDEX idx_insights_brand_category ON insights (brand_id, category);
```

Equivalent Drizzle definition (source of truth in this codebase — types are inferred from it):

```ts
export const insights = pgTable("insights", {
  id:            uuid("id").primaryKey().defaultRandom(),
  brandId:       uuid("brand_id").notNull().references(() => brands.id),
  uploadBatchId: uuid("upload_batch_id").notNull().references(() => uploadBatches.id),
  category:      insightCategoryEnum("category").notNull(),
  severity:      insightSeverityEnum("severity").notNull(),
  title:         text("title").notNull(),
  summary:       text("summary"),
  detail:        jsonb("detail"),
  metricValue:   real("metric_value"),
  metricUnit:    varchar("metric_unit", { length: 20 }),
  createdAt:     timestamp("created_at").defaultNow().notNull(),
}, (table) => [
  index("idx_insights_brand_category").on(table.brandId, table.category),
]);
```

The `(brand_id, category)` index supports the UI's "show me all insights of type X for this tenant" query and the summary pass's `find(i => i.category === …)` lookups.

---

## 8. Dependencies

**npm**

| Package | Role |
| --- | --- |
| `@anthropic-ai/sdk` | Claude API client — used by the waste pass (enrichment) and the executive-summary pass. |
| `drizzle-orm` | Typed SQL builder; `$inferInsert` gives the canonical `InsightRow` type so passes and DB never drift. |
| `next` | API route handler (`route.ts`) — any HTTP framework works; the engine is framework-agnostic. |
| PostgreSQL driver | e.g. `postgres` / `pg` — the engine assumes Postgres (enums, JSONB, `gen_random_uuid()`). |

**Environment variables**

| Var | Purpose |
| --- | --- |
| `ANTHROPIC_API_KEY` | Auth for the Claude calls in the LLM-enabled passes. |
| `DATABASE_URL` (or your driver's equivalent) | Postgres connection used by `db`. |

Model used: `claude-sonnet-4-6` (swap freely; the prompts are model-agnostic).

---

## 9. Porting Checklist

**Swap the domain passes. Keep the framework.** The orchestrator, the pass contract, the `insights` table, the batch-insert pattern, and the executive-summary pass are domain-agnostic — port them as-is. Only the analysis logic inside each pass changes.

- [ ] **Keep** `engine.ts` (orchestrator): delete-old → run passes → concat → summary last → batch insert.
- [ ] **Keep** the `Pass` contract: `(scopeId, batchId) => Promise<InsightRow[]>`, pure (no writes), returns an array.
- [ ] **Keep** the `insights` table shape (`category`, `severity`, `title`, `summary`, `detail` JSONB, `metric_value`, `metric_unit`) and the composite index.
- [ ] **Keep** the executive-summary pass structure: read prior outputs → build metrics context → LLM → JSON-parse defensively → fallback insight on failure.
- [ ] **Rename the scope key** `brand_id` → your tenant/account/user key throughout.
- [ ] **Rewrite the `category` enum** with your own pass names (e.g. `retention_cohort`, `dormant_users`, `revenue_anomaly`).
- [ ] **Replace each domain pass** with your own queries + thresholds. Mental substitution: *campaigns → users*, *ROAS → retention*, *wasted spend → failed jobs*, *kill list → at-risk accounts*.
- [ ] **Tune the threshold ladders** (`< 5 critical`, `> 15 scale`, `> 20000 waste`) to your domain's meaningful breakpoints.
- [ ] **Rewrite the summary prompt** for your analyst persona and the verdicts your product reports.
- [ ] **Wire the trigger**: an authenticated route (or a queue/cron job) that calls `generateInsights(scopeId, batchId)` after each data ingest.
- [ ] **Set** `ANTHROPIC_API_KEY` and your DB connection env var.
- [ ] **(Optional)** parallelize passes 1..N with `Promise.all`; the summary pass must still run last.
- [ ] **(Optional)** swap delete-then-insert for append-only history if you want trend tracking across runs.

---

*Built by [Quantana](https://quantana.in). MIT licensed.*
