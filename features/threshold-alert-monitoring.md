# Threshold-Based Alert & Anomaly Monitoring

> A portable pattern for turning raw time-series metrics into actionable, severity-scored alerts. Aggregate your metrics over a window, compare them against per-entity configurable thresholds stored as JSONB, and surface the breaches — ranked by severity — through any API endpoint or dashboard.
>
> The ad-tech metrics shown here (zero-conversion burn, ROAS drops, budget pacing) are just examples. The reusable core is the generic **aggregate → compare-to-threshold → severity-score → surface** loop, with thresholds you can tune per tenant/entity without code changes.

## 1. Overview

`<YourApp>` ingests raw metric rows into a `dailyMetrics`-style table (one row per entity per day). On every dashboard load, the alert generator:

1. Runs windowed SQL aggregations (yesterday, last 7 days, last 14 days, month-to-date).
2. Compares each aggregate against thresholds that live in a JSONB column on a per-entity config table (`agentConfig.alertThresholds`).
3. Emits structured `Alert` objects tagged with a `type`, a `severity` (`high` / `medium` / `low`), a human-readable `message`, and the offending entity.
4. Returns the alert array inline with the rest of the dashboard payload — no separate polling endpoint, no background job required.

Because thresholds are data (not constants), each tenant/entity can set its own monthly budget, drop tolerance, and pacing limits. The same generator code serves everyone.

## 2. Flow

```
                          ┌──────────────────────────────┐
                          │   raw metric rows             │
                          │   (dailyMetrics: 1 row/       │
                          │    entity/day)                │
                          └───────────────┬──────────────┘
                                          │
                  ┌───────────────────────┼───────────────────────┐
                  │ QUERY (windowed)      │                       │
                  ▼                       ▼                       ▼
          ┌──────────────┐       ┌──────────────┐        ┌──────────────┐
          │  yesterday   │       │ this 7d /    │        │ month-to-date│
          │              │       │ last 7d      │        │              │
          └──────┬───────┘       └──────┬───────┘        └──────┬───────┘
                 │                      │                       │
                 ▼                      ▼                       ▼
          ┌──────────────────────────────────────────────────────────┐
          │  AGGREGATE  (SQL: sum / group by entity)                  │
          └───────────────────────────┬──────────────────────────────┘
                                       │
                                       ▼
          ┌──────────────────────────────────────────────────────────┐
          │  LOAD THRESHOLDS                                          │
          │  agentConfig.alertThresholds  (JSONB, per entity)        │
          └───────────────────────────┬──────────────────────────────┘
                                       │
                                       ▼
          ┌──────────────────────────────────────────────────────────┐
          │  COMPARE  (aggregate  vs  threshold)                      │
          │   • absolute     (spend > X)                             │
          │   • delta/ratio  (week-over-week drop %)                 │
          │   • projection   (run-rate vs budget)                    │
          └───────────────────────────┬──────────────────────────────┘
                                       │
                                       ▼
          ┌──────────────────────────────────────────────────────────┐
          │  SEVERITY-SCORE  → Alert{type, severity, message, entity} │
          └───────────────────────────┬──────────────────────────────┘
                                       │
                                       ▼
          ┌──────────────────────────────────────────────────────────┐
          │  SURFACE  → GET /api/.../dashboard  → { ..., alerts: [] } │
          └──────────────────────────────────────────────────────────┘
```

## 3. Threshold Configuration

Thresholds are stored as a JSONB column on a per-entity config table, with a sensible default object so new entities get monitoring out of the box. Operators tune these per tenant without redeploying.

```ts
// src/lib/db/schema.ts
export const agentConfig = pgTable("agent_config", {
  id: uuid("id").primaryKey().defaultRandom(),
  brandId: uuid("brand_id").notNull().references(() => brands.id),
  platform: platformEnum("platform").notNull(),
  trustLevel: integer("trust_level").default(1),
  maxDailyBudgetChangePct: real("max_daily_budget_change_pct").default(10),
  maxBidChangePct: real("max_bid_change_pct").default(20),
  alertThresholds: jsonb("alert_thresholds").default({
    zeroConvBurnThreshold: 1000, roasDropPct: 30, budgetPacingOvershootPct: 15,
    negativeMarginDays: 3, agencyDriftGap: 1.5, highPerformerRoas: 2.5, highPerformerDays: 7,
  }),
  actionOverrides: jsonb("action_overrides").default({}),
}, (t) => ({
  brandPlatformUniq: unique().on(t.brandId, t.platform),
}));
```

Key points for porting:

- One config row **per entity** (here `(brandId, platform)`, enforced unique). Swap this for whatever your "monitored entity" is.
- `alertThresholds` is freeform JSONB — add/remove keys as your metric set evolves; no migration needed.
- Per-entity thresholds (e.g. `monthlyBudget`) can be read at compare time. Numbers absent from the JSONB simply skip that check.

## 4. The Alert Generator

`src/lib/alerts.ts` — the entire engine. Each check is an independent block: **query window → aggregate in SQL → compare to threshold → push a severity-scored `Alert`.**

```ts
import { db } from "./db";
import { dailyMetrics, platformConnections, agentConfig } from "./db/schema";
import { eq, and, gte, sql } from "drizzle-orm";

export interface Alert {
  type: "zero_conv_burn" | "roas_drop" | "budget_pacing";
  severity: "high" | "medium" | "low";
  message: string;
  platform: string;       // the offending entity
  campaign?: string;      // optional sub-entity
}

export async function generateAlerts(brandId: string): Promise<Alert[]> {
  const alerts: Alert[] = [];
  const yesterday = new Date(Date.now() - 86400000).toISOString().split("T")[0];
  const sevenDaysAgo = new Date(Date.now() - 7 * 86400000).toISOString().split("T")[0];
  const fourteenDaysAgo = new Date(Date.now() - 14 * 86400000).toISOString().split("T")[0];

  // 1. ABSOLUTE THRESHOLD — zero-conversion burn:
  //    aggregate spend per campaign for yesterday; flag if spend > X and conversions == 0
  const zeroBurnCampaigns = await db
    .select({
      campaignName: dailyMetrics.campaignName,
      platform: platformConnections.platform,
      spend: sql<number>`sum(${dailyMetrics.spend})`,
    })
    .from(dailyMetrics)
    .innerJoin(platformConnections, eq(dailyMetrics.platformConnectionId, platformConnections.id))
    .where(and(eq(dailyMetrics.brandId, brandId), gte(dailyMetrics.date, yesterday)))
    .groupBy(dailyMetrics.campaignName, platformConnections.platform)
    .having(and(sql`sum(${dailyMetrics.conversions}) = 0`, sql`sum(${dailyMetrics.spend}) > 1000`));

  for (const c of zeroBurnCampaigns) {
    alerts.push({
      type: "zero_conv_burn",
      severity: "high",                          // wasted spend → highest severity
      message: `"${c.campaignName}" spent ${formatINR(c.spend)} with 0 conversions`,
      platform: c.platform,
      campaign: c.campaignName || undefined,
    });
  }

  // 2. DELTA / RATIO THRESHOLD — week-over-week ROAS drop:
  //    aggregate two adjacent 7-day windows, compute ratio, flag if drop% > threshold
  const thisWeek = await db
    .select({ platform: platformConnections.platform, spend: sql<number>`sum(${dailyMetrics.spend})`, revenue: sql<number>`sum(${dailyMetrics.revenue})` })
    .from(dailyMetrics)
    .innerJoin(platformConnections, eq(dailyMetrics.platformConnectionId, platformConnections.id))
    .where(and(eq(dailyMetrics.brandId, brandId), gte(dailyMetrics.date, sevenDaysAgo)))
    .groupBy(platformConnections.platform);

  const lastWeek = await db
    .select({ platform: platformConnections.platform, spend: sql<number>`sum(${dailyMetrics.spend})`, revenue: sql<number>`sum(${dailyMetrics.revenue})` })
    .from(dailyMetrics)
    .innerJoin(platformConnections, eq(dailyMetrics.platformConnectionId, platformConnections.id))
    .where(and(eq(dailyMetrics.brandId, brandId), gte(dailyMetrics.date, fourteenDaysAgo), sql`${dailyMetrics.date} < ${sevenDaysAgo}`))
    .groupBy(platformConnections.platform);

  for (const tw of thisWeek) {
    const lw = lastWeek.find((l) => l.platform === tw.platform);
    if (!lw || lw.spend === 0 || tw.spend === 0) continue;   // guard divide-by-zero
    const thisRoas = tw.revenue / tw.spend;
    const lastRoas = lw.revenue / lw.spend;
    if (lastRoas > 0) {
      const dropPct = ((lastRoas - thisRoas) / lastRoas) * 100;
      if (dropPct > 30) {
        alerts.push({ type: "roas_drop", severity: "medium", message: `${tw.platform} ROAS dropped ${Math.round(dropPct)}% week-over-week`, platform: tw.platform });
      }
    }
  }

  // 3. PROJECTION THRESHOLD — budget pacing:
  //    aggregate month-to-date spend, project a run-rate to month end,
  //    compare against the PER-ENTITY threshold pulled from JSONB config
  const dayOfMonth = new Date().getDate();
  const daysInMonth = new Date(new Date().getFullYear(), new Date().getMonth() + 1, 0).getDate();
  const monthStart = new Date(new Date().getFullYear(), new Date().getMonth(), 1).toISOString().split("T")[0];

  const monthlySpend = await db
    .select({ platform: platformConnections.platform, totalSpend: sql<number>`coalesce(sum(${dailyMetrics.spend}), 0)` })
    .from(dailyMetrics)
    .innerJoin(platformConnections, eq(dailyMetrics.platformConnectionId, platformConnections.id))
    .where(and(eq(dailyMetrics.brandId, brandId), gte(dailyMetrics.date, monthStart)))
    .groupBy(platformConnections.platform);

  const configs = await db.query.agentConfig.findMany({ where: eq(agentConfig.brandId, brandId) });

  for (const ms of monthlySpend) {
    const config = configs.find((c) => c.platform === ms.platform);
    if (!config?.alertThresholds) continue;
    const thresholds = config.alertThresholds as any;     // ← per-entity JSONB thresholds
    const monthlyBudget = thresholds.monthlyBudget;
    if (!monthlyBudget || monthlyBudget <= 0) continue;   // unset threshold → skip check
    const projectedSpend = (ms.totalSpend / dayOfMonth) * daysInMonth;
    const overshootPct = ((projectedSpend - monthlyBudget) / monthlyBudget) * 100;
    if (overshootPct > 15) {
      alerts.push({ type: "budget_pacing", severity: "medium", message: `${ms.platform} projected to exceed monthly budget by ${Math.round(overshootPct)}% (${formatINR(projectedSpend)} vs ${formatINR(monthlyBudget)})`, platform: ms.platform });
    }
  }

  return alerts;
}

function formatINR(n: number) {
  return `₹${Math.round(n).toLocaleString("en-IN")}`;
}
```

### The three comparison archetypes (reuse these)

| Archetype | Logic | Example check | Severity |
|-----------|-------|---------------|----------|
| **Absolute** | `aggregate > X` (often as SQL `HAVING`) | zero-conv burn (spend > 1000, conv = 0) | `high` |
| **Delta / ratio** | compare two windows, flag `drop% > X` | ROAS dropped > 30% WoW | `medium` |
| **Projection** | run-rate to period end vs configured budget | spend projected > budget + 15% | `medium` |

Severity is assigned statically per check based on business impact (wasted money = `high`; efficiency erosion / pacing = `medium`). To make severity dynamic, scale it off the breach magnitude (e.g. `overshootPct > 50 ? "high" : "medium"`).

## 5. Surfacing Alerts

Alerts are computed inside the dashboard aggregator and returned inline — there is no separate alerts endpoint or queue.

`src/lib/metrics.ts` (`getDashboardData`) calls the generator and folds the result into the payload:

```ts
import { generateAlerts } from "./alerts";

export async function getDashboardData(brandId: string) {
  // ...aggregate today's metrics, platform breakdown, efficiency score...
  const alerts = await generateAlerts(brandId);
  return {
    today: todayMetrics[0] || { /* zeros */ },
    efficiencyScore,
    platformBreakdown,
    alerts,                       // ← alert array rides along with the dashboard
  };
}
```

The HTTP route is a thin auth-guarded wrapper — `src/app/api/brands/[brandId]/dashboard/route.ts`:

```ts
export async function GET(
  req: NextRequest,
  { params }: { params: Promise<{ brandId: string }> }
) {
  const { brandId } = await params;
  const access = await requireBrandAccess(brandId);   // tenant authorization
  if ("error" in access) return access.error;
  const data = await getDashboardData(brandId);       // includes alerts
  return NextResponse.json(data);
}
```

Result: `GET /api/brands/:brandId/dashboard` returns `{ today, efficiencyScore, platformBreakdown, alerts: Alert[] }`. The client renders the `alerts` array, typically sorted/colored by `severity`.

## 6. Database Schema

Only two tables matter for porting: the **metric store** (source of aggregates) and the **per-entity config** (threshold storage). Alerts themselves are computed on demand and **not persisted** — if you need history, add an `alerts` table and insert from the generator.

```ts
// METRIC STORE — one row per entity per day
export const dailyMetrics = pgTable("daily_metrics", {
  id: uuid("id").primaryKey().defaultRandom(),
  brandId: uuid("brand_id").notNull().references(() => brands.id),
  platformConnectionId: uuid("platform_connection_id").notNull().references(() => platformConnections.id),
  date: date("date").notNull(),
  campaignName: varchar("campaign_name", { length: 500 }),
  spend: real("spend").default(0),
  impressions: integer("impressions").default(0),
  clicks: integer("clicks").default(0),
  conversions: integer("conversions").default(0),
  revenue: real("revenue").default(0),
  // ...other metric columns...
  createdAt: timestamp("created_at").defaultNow().notNull(),
});

// PER-ENTITY THRESHOLD CONFIG — JSONB thresholds, unique per entity
export const agentConfig = pgTable("agent_config", {
  id: uuid("id").primaryKey().defaultRandom(),
  brandId: uuid("brand_id").notNull().references(() => brands.id),
  platform: platformEnum("platform").notNull(),
  alertThresholds: jsonb("alert_thresholds").default({
    zeroConvBurnThreshold: 1000, roasDropPct: 30, budgetPacingOvershootPct: 15,
    negativeMarginDays: 3, agencyDriftGap: 1.5, highPerformerRoas: 2.5, highPerformerDays: 7,
  }),
  // ...other config columns...
}, (t) => ({
  brandPlatformUniq: unique().on(t.brandId, t.platform),
}));
```

> **Optional — persisting alerts.** To keep an audit trail, define `alerts(id, brandId, type, severity, message, entity, createdAt, resolvedAt)` and `db.insert(alerts)` inside `generateAlerts` instead of (or in addition to) returning them. Add a uniqueness/dedup guard so the same breach isn't re-inserted on every dashboard load.

## 7. Dependencies

**npm**

```jsonc
{
  "dependencies": {
    "drizzle-orm": "^0.30",      // query builder + sql`` aggregation
    "pg": "^8",                   // (or postgres) Postgres driver
    "next": "^14"                 // only for the API route wrapper; swap for any HTTP layer
  },
  "devDependencies": {
    "drizzle-kit": "^0.21"        // migrations for the two tables above
  }
}
```

The generator itself depends only on `drizzle-orm` and a Postgres connection — it is framework-agnostic. Next.js is incidental to the surfacing layer.

**Environment variables**

| Var | Purpose |
|-----|---------|
| `DATABASE_URL` | Postgres connection string used by the Drizzle `db` client (`src/lib/db`). |

No external services, queues, or cron are required: alerts are derived synchronously from the metric store on each read.

## 8. Porting Checklist

- [ ] **Define your own metrics.** Replace `dailyMetrics` columns (`spend`, `conversions`, `revenue`, …) with your domain's measurements. Keep "one row per entity per time-bucket."
- [ ] **Define your own entity.** Swap `(brandId, platform)` for your monitored unit (user, device, service, region, …) and keep the unique constraint on the config table.
- [ ] **Define your own thresholds.** Edit the `alertThresholds` JSONB default; add/remove keys freely. Decide which are global constants in code vs per-entity values read from JSONB (like `monthlyBudget`).
- [ ] **Pick comparison archetypes.** For each metric, choose absolute / delta-ratio / projection (Section 4 table) and write one independent block per check.
- [ ] **Assign severities.** Map each check to `high` / `medium` / `low` by business impact; optionally scale severity by breach magnitude.
- [ ] **Guard the math.** Skip divide-by-zero (empty windows), and `continue` when a threshold is unset so partial config never crashes the loop.
- [ ] **Wire into your read path.** Call `generateAlerts(entityId)` from your dashboard/data aggregator and return the array inline; add tenant authorization on the route.
- [ ] **(Optional) Persist + dedup** alerts if you need history, notifications, or resolution tracking.
- [ ] **Tune windows** (yesterday / 7d / 14d / MTD) to match your data cadence.

---

*Built by [Quantana](https://quantana.in). MIT licensed.*
