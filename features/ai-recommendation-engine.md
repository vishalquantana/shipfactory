# AI Recommendation Engine

> A domain-agnostic framework for turning raw data into reviewable, actionable recommendations. Multiple LLM analysis passes run concurrently over the same dataset, each producing typed suggestions; conflicting/duplicate recommendations are de-duplicated, every recommendation carries a confidence and a risk score, and a human approve/reject/modify workflow gates what actually happens. Every decision is recorded so the system learns each account's preferences over time.

---

## 1. Overview

The engine separates **what could be done** (LLM analysis) from **what should be done** (human approval) from **what the user actually tends to accept** (behavioral learning). It works for any domain where an LLM can look at structured data and propose discrete, typed actions.

Three moving parts:

1. **Analyzers** — N independent prompts, each focused on one "lens" over the same dataset, run concurrently. Each returns a JSON array of typed recommendation objects (`actionType`, `confidence`, `risk`, `reasoning`, `projectedImpact`).
2. **Recommender** — persists recommendations, dropping any that conflict with an already-pending one for the same target + action type.
3. **Human-in-the-loop + Learning** — single and batch approve/reject/modify endpoints flip recommendation status and feed every decision into a per-account preference table that the analyzers read back as context on the next run.

The ad-tech action types in this document (`bid_adjust`, `budget_change`, `pause`, …) are **examples**. Swap them for your domain's actions — the framework is unchanged.

> Naming note: this spec abstracts the reference implementation. `<YourApp>` = the product, `user` = the human reviewer, `account` = the tenant whose data is analyzed. In the source these map to *Ladya*, *founder*, and *brand*.

---

## 2. Architecture

```
                       ┌────────────────────────────────────────────┐
                       │              account data store             │
                       │        (metrics, events, entities …)        │
                       └───────────────────────┬────────────────────┘
                                               │  pull window + history
                                               ▼
                ┌──────────────────────────────────────────────────────────┐
                │                analyzeAll(accountId)                       │
                │   fan-out: N concurrent LLM analysis passes (Promise.all)  │
                │                                                            │
                │  ┌──────────┐ ┌──────────┐ ┌──────────┐      ┌──────────┐ │
                │  │ pass A   │ │ pass B   │ │ pass C   │  ...  │ pass N   │ │
                │  │ (lens 1) │ │ (lens 2) │ │ (lens 3) │      │ (lens N) │ │
                │  └────┬─────┘ └────┬─────┘ └────┬─────┘      └────┬─────┘ │
                │       └───────┬────┴────────────┴─────────────────┘       │
                └───────────────┼──────────────────────────────────────────┘
                                ▼
                  flat list of typed AnalysisResult[]
                  { actionType, confidence, risk, reasoning, projectedImpact }
                                │
                                ▼
                ┌──────────────────────────────────────────┐
                │        saveRecommendations()              │
                │  conflict dedup: skip if a PENDING rec     │
                │  already exists for same target+action     │
                └───────────────────┬──────────────────────┘
                                    ▼
                ┌──────────────────────────────────────────┐
                │     recommendations table (status=pending) │
                └───────────────────┬──────────────────────┘
                                    ▼
          ┌──────────────── human-in-the-loop API ─────────────────┐
          │   GET  /recommendations            (list / filter)      │
          │   POST /recommendations/:id/approve                     │
          │   POST /recommendations/:id/reject                      │
          │   POST /recommendations/:id/modify (edit + approve)     │
          │   POST /recommendations/batch      (bulk approve/reject)│
          └───────────────────┬───────────────────────────────────┘
                              ▼
                   recordUserAction(account, target, actionType, decision)
                              │
                              ▼
          ┌──────────────────────────────────────────────────────┐
          │   user_preferences  (approvalRate per action+target)  │
          └───────────────────┬──────────────────────────────────┘
                              │  read back as "historical outcomes"
                              └────────────►  next analyzeAll() run
                                              (the learning loop)
```

---

## 3. The Analyzer Passes

Each analysis "lens" is just a prompt. They share one strict system prompt that pins the **output contract** (a JSON array of typed objects), and they all run against the same windowed dataset plus recent decision history. Because the passes are independent, they fan out concurrently.

```ts
// agent/analyzer.ts
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

// One prompt per "lens". Swap these for your domain — the framework is unchanged.
const ANALYSIS_PROMPTS: Record<AnalysisType, string> = {
  spend_efficiency:
    "Analyze these campaigns for spend efficiency. Find campaigns with high spend but low ROAS (below 1.5x). " +
    "Suggest bid reductions for underperformers. Only flag campaigns with at least ₹500 spend.",
  zero_conv_burn:
    "Find campaigns that are spending money but generating zero conversions over the last 48 hours. " +
    "Suggest pausing campaigns with >₹1000 spend and 0 conversions.",
  roas_trends:
    "Compare this week's ROAS to last week's for each campaign. " +
    "Flag campaigns where ROAS has dropped more than 30% week-over-week.",
  // budget_pacing, cross_platform_arbitrage, high_performers …
};

// The output contract every pass must honor.
const SYSTEM_PROMPT = `You are an analyst. Respond ONLY with a JSON array of
recommendation objects. No markdown, no explanation.

Each object must have:
- action_type: one of "bid_adjust", "budget_change", "pause", "resume", "keyword_add", "keyword_remove"
- action_payload: { field, current_value, suggested_value, change_pct }
- confidence: number 0-1
- risk: "low" | "medium" | "high"
- reasoning: string (1-2 sentences)
- projected_impact: string

If no recommendations for this analysis type, return an empty array: []`;

export async function runAnalysis(
  accountId: string,
  analysisType: AnalysisType,
): Promise<AnalysisResult[]> {
  // 1. Pull the analysis window (e.g. last 7 days of metrics)
  const metrics = await loadWindow(accountId);
  if (metrics.length === 0) return [];

  // 2. Pull recent decision history — this is the learning loop's read side
  const history = await loadDecisionHistory(accountId, { limit: 20 });

  const userPrompt = `${ANALYSIS_PROMPTS[analysisType]}

Here is the data for the last 7 days:
${JSON.stringify(metrics, null, 2)}

${history.length > 0
  ? `Historical outcomes for this account:\n${JSON.stringify(history, null, 2)}`
  : "No historical outcome data yet."}`;

  const response = await client.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 4096,
    system: SYSTEM_PROMPT,
    messages: [{ role: "user", content: userPrompt }],
  });

  const text = response.content[0].type === "text" ? response.content[0].text : "";

  // 3. Parse + validate into typed results. Drop anything malformed.
  try {
    const parsed = JSON.parse(text);
    if (!Array.isArray(parsed)) return [];

    return parsed
      .filter((item) => item.action_type && item.confidence != null)
      .map((item) => ({
        actionType: item.action_type as AnalysisResult["actionType"],
        actionPayload: item.action_payload as AnalysisResult["actionPayload"],
        confidence: item.confidence as number,
        risk: item.risk as AnalysisResult["risk"],
        reasoning: item.reasoning as string,
        projectedImpact: item.projected_impact as string,
        platform: item.platform as string,
      }));
  } catch {
    console.error(`[Analyzer] Failed to parse response for ${analysisType}`);
    return [];   // one bad pass never breaks the batch
  }
}

const ALL_ANALYSIS_TYPES: AnalysisType[] = [
  "spend_efficiency", "zero_conv_burn", "roas_trends",
  "budget_pacing", "cross_platform_arbitrage", "high_performers",
];

// Fan out all lenses concurrently and flatten. Each pass is isolated, so a
// single failed/empty pass contributes nothing rather than aborting the run.
export async function analyzeAll(accountId: string): Promise<AnalysisResult[]> {
  const settled = await Promise.all(
    ALL_ANALYSIS_TYPES.map((analysisType) =>
      runAnalysis(accountId, analysisType)
        .then((results) => {
          console.log(`[Analyzer] ${analysisType}: ${results.length} recommendations`);
          return results;
        })
        .catch((err) => {
          console.error(`[Analyzer] Error in ${analysisType}:`, err);
          return [] as AnalysisResult[];
        }),
    ),
  );

  return settled.flat();
}
```

**Why this shape works for any domain:**

- The **system prompt is the contract.** Every pass, regardless of lens, must emit the same object shape. That's what makes the downstream pipeline domain-agnostic.
- **Passes are independent.** Adding a new lens = adding one entry to `ANALYSIS_PROMPTS` + `ALL_ANALYSIS_TYPES`. No other code changes.
- **Failure is contained.** Each pass `.catch`es to `[]`, so a single rate-limit or parse error never aborts the whole batch.
- **`Promise.all` fan-out** turns N sequential round-trips into one wall-clock latency. (The reference repo runs them in a `for` loop; concurrency is the recommended portable form shown here.)

---

## 4. Conflict Detection & Scoring

Multiple lenses inspecting the same data will sometimes propose the *same* action against the *same* target — e.g. both `spend_efficiency` and `roas_trends` want to cut the bid on one campaign. The recommender de-duplicates at persist time: **if a `pending` recommendation already exists for the same target + `actionType`, the new one is skipped.**

```ts
// agent/recommender.ts
export async function saveRecommendations(
  accountId: string,
  results: AnalysisResult[],
): Promise<string[]> {
  const savedIds: string[] = [];

  for (const result of results) {
    // Conflict detection: is there already a PENDING rec for this
    // target + action type? If so, this one is a duplicate/conflict — skip it.
    const existing = await db
      .select({ id: recommendations.id, actionPayload: recommendations.actionPayload })
      .from(recommendations)
      .where(
        and(
          eq(recommendations.brandId, accountId),
          eq(recommendations.actionType, result.actionType),
          eq(recommendations.status, "pending"),
        ),
      );

    const hasConflict = existing.some((rec) => {
      const payload = rec.actionPayload as Record<string, unknown> | null;
      return payload?.campaignName === result.campaignName; // same target
    });

    if (hasConflict) {
      console.log(`[Recommender] Skipping conflicting ${result.actionType} for ${result.campaignName}`);
      continue;
    }

    // Persist with confidence + risk + reasoning + projected impact intact.
    const [inserted] = await db
      .insert(recommendations)
      .values({
        brandId: accountId,
        platform: result.platform as "instamart" | "zepto" | "blinkit",
        actionType: result.actionType,
        actionPayload: result.actionPayload,
        confidence: result.confidence,        // 0..1, from the LLM
        risk: result.risk,                    // "low" | "medium" | "high"
        reasoning: result.reasoning,
        projectedImpact: result.projectedImpact,
        status: "pending",
      })
      .returning({ id: recommendations.id });

    savedIds.push(inserted.id);
  }

  console.log(`[Recommender] Saved ${savedIds.length} of ${results.length} recommendations`);
  return savedIds;
}
```

**Scoring fields** (carried straight through from the analyzer to the DB and into the UI):

| Field | Type | Meaning |
|---|---|---|
| `confidence` | `real` (0–1) | How sure the analyzer is. Drives sort order and auto-approve eligibility. |
| `risk` | `"low" \| "medium" \| "high"` | Blast radius if the action is wrong. Gates which recs a human must see. |
| `reasoning` | `text` | 1–2 sentence justification shown to the reviewer. |
| `projectedImpact` | `text` | Expected outcome, e.g. *"ROAS improvement from 1.2x to 2.0x"*. |

> **Conflict-dedup is intentionally simple and pluggable.** The default key is `(target, actionType, status=pending)`. Tighten it (compare payloads, keep the higher-confidence rec, mark the loser `conflicted`) or loosen it to fit your domain — the surrounding pipeline doesn't change.

---

## 5. Approve / Reject / Modify Workflow

Nothing acts automatically by default — every recommendation lands as `pending` and waits for a human. There are four endpoints: list/filter, single approve, single reject, modify-then-approve, plus a bulk endpoint. **Every one of them records the decision for learning (Section 6).**

**List & filter** — `GET /api/accounts/:accountId/recommendations`

```ts
const url = new URL(req.url);
const status = url.searchParams.get("status");
const platform = url.searchParams.get("platform");
const risk = url.searchParams.get("risk");

const conditions = [eq(recommendations.brandId, accountId)];
if (status) conditions.push(eq(recommendations.status, status));
if (platform) conditions.push(eq(recommendations.platform, platform));

const rows = await db
  .select().from(recommendations)
  .where(and(...conditions))
  .orderBy(desc(recommendations.createdAt))
  .limit(100);

// risk is a varchar, not an enum → filter in memory
const filtered = risk ? rows.filter((r) => r.risk === risk) : rows;
return NextResponse.json(filtered);
```

**Approve** — `POST /recommendations/:id/approve`

```ts
const [rec] = await db.select().from(recommendations)
  .where(and(eq(recommendations.id, id), eq(recommendations.brandId, accountId)));

if (!rec) return NextResponse.json({ error: "Not found" }, { status: 404 });
if (rec.status !== "pending" && rec.status !== "expired") {
  return NextResponse.json({ error: "Cannot approve — status is " + rec.status }, { status: 400 });
}

await db.update(recommendations)
  .set({ status: "approved" })
  .where(eq(recommendations.id, id));

await recordUserAction(accountId, rec.platform, rec.actionType, "approved"); // ← learning
return NextResponse.json({ success: true });
```

**Reject** — `POST /recommendations/:id/reject` (same shape, sets `status: "rejected"` and records `"rejected"`).

**Modify** — `POST /recommendations/:id/modify` — the reviewer edits the payload, and the edit counts as an approval of a *changed* action:

```ts
const body = await req.json();
if (!body.actionPayload) {
  return NextResponse.json({ error: "actionPayload required" }, { status: 400 });
}

await db.update(recommendations)
  .set({ status: "approved", actionPayload: body.actionPayload }) // approve the edited version
  .where(eq(recommendations.id, id));

await recordUserAction(accountId, rec.platform, rec.actionType, "modified"); // ← distinct signal
```

**Batch** — `POST /recommendations/batch` — approve/reject many at once by `ids` or by `filter`:

```ts
const { action, ids, filter } = body;          // action: "approve" | "reject"

let targetRecs;
if (ids?.length) {
  targetRecs = await db.select().from(recommendations).where(and(
    eq(recommendations.brandId, accountId),
    inArray(recommendations.id, ids),
    eq(recommendations.status, "pending"),
  ));
} else if (filter) {
  const conditions = [eq(recommendations.brandId, accountId), eq(recommendations.status, "pending")];
  if (filter.platform) conditions.push(eq(recommendations.platform, filter.platform));
  targetRecs = await db.select().from(recommendations).where(and(...conditions));
  if (filter.risk) targetRecs = targetRecs.filter((r) => r.risk === filter.risk);
}

const newStatus     = action === "approve" ? "approved" : "rejected";
const userDecision  = action === "approve" ? "approved" : "rejected";

for (const rec of targetRecs) {
  await db.update(recommendations).set({ status: newStatus }).where(eq(recommendations.id, rec.id));
  await recordUserAction(accountId, rec.platform, rec.actionType, userDecision); // ← every item logged
}

return NextResponse.json({ success: true, count: targetRecs.length });
```

Status lifecycle: `pending → approved | rejected | modified(→approved)`; downstream execution can later move `approved → executed | execution_failed`, and unactioned recs age out to `expired`.

---

## 6. Behavioral Learning

Every decision the user makes is folded into a per-account, per-(target-class, action-type) preference row. Over time these rows describe *what this account actually accepts* — which the analyzer reads back as "historical outcomes" (Section 3), and which can gate auto-approval once the approval rate is high enough.

```ts
// src/lib/user-actions.ts  (founder-actions.ts in the reference impl)
export async function recordUserAction(
  accountId: string,
  platform: string,        // the target class / dimension you bucket by
  actionType: string,
  action: "approved" | "rejected" | "modified",
): Promise<void> {
  const existing = await db
    .select().from(userPreferences)
    .where(and(
      eq(userPreferences.brandId, accountId),
      eq(userPreferences.platform, platform),
      eq(userPreferences.actionType, actionType),
    ))
    .limit(1);

  if (existing.length === 0) {
    // First decision for this (account, target, action) — seed the row.
    await db.insert(userPreferences).values({
      brandId: accountId,
      platform,
      actionType,
      approvedCount: action === "approved" ? 1 : 0,
      rejectedCount: action === "rejected" ? 1 : 0,
      modifiedCount: action === "modified" ? 1 : 0,
      approvalRate:  action === "approved" ? 1 : 0,
    });
  } else {
    // Increment the matching counter and recompute approval rate.
    const pref = existing[0];
    const newApproved = (pref.approvedCount ?? 0) + (action === "approved" ? 1 : 0);
    const newRejected = (pref.rejectedCount ?? 0) + (action === "rejected" ? 1 : 0);
    const newModified = (pref.modifiedCount ?? 0) + (action === "modified" ? 1 : 0);
    const total       = newApproved + newRejected + newModified;
    const newRate     = total > 0 ? newApproved / total : 0;

    await db.update(userPreferences)
      .set({
        approvedCount: newApproved,
        rejectedCount: newRejected,
        modifiedCount: newModified,
        approvalRate:  newRate,
      })
      .where(eq(userPreferences.id, pref.id));
  }
}
```

**How the loop closes:**

- `approvalRate` is a running ratio per `(account, target-class, actionType)`. A consistently-approved action type can flip `autoApproveEligible = true`, letting low-risk recs skip the queue.
- `modifiedCount` is a distinct signal from `approved` — a high modify rate means "the user wants this *kind* of action but never with the magnitude the analyzer suggests," which is a prompt-tuning hint.
- These rows (and the richer per-decision `learning_log`) are exactly what `runAnalysis` passes back into the model as *"Historical outcomes for this account."* The model sees what got rejected last time and stops proposing it.

> The learning store is deliberately a **counter table, not a model.** No retraining, no embeddings — just bucketed approval rates that are cheap to write on every decision and cheap to read as prompt context. That keeps the loop online and explainable.

---

## 7. Database Schema

```sql
-- The core table: one row per proposed action.
CREATE TYPE platform AS ENUM ('instamart','zepto','blinkit','amazon','flipkart','meta','google'); -- target dimension; rename per domain
CREATE TYPE action_type AS ENUM ('bid_adjust','budget_change','pause','resume','keyword_add','keyword_remove'); -- EXAMPLE actions
CREATE TYPE recommendation_status AS ENUM ('pending','approved','rejected','executed','expired','conflicted','execution_failed');
CREATE TYPE outcome_verdict AS ENUM ('improved','neutral','degraded');

CREATE TABLE recommendations (
  id                     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  brand_id               UUID NOT NULL REFERENCES brands(id),   -- account_id
  platform               platform NOT NULL,                     -- target dimension
  type                   VARCHAR(100),                          -- analysis lens, optional
  action_type            action_type NOT NULL,
  action_payload         JSONB,                                 -- { field, current_value, suggested_value, change_pct, ... }
  projected_impact       TEXT,                                  -- "ROAS 1.2x -> 2.0x"
  confidence             REAL,                                  -- 0..1
  risk                   VARCHAR(20),                           -- low | medium | high
  reasoning              TEXT,
  status                 recommendation_status DEFAULT 'pending',
  auto_executed          BOOLEAN DEFAULT FALSE,
  executed_at            TIMESTAMP,
  execution_proof_s3_key TEXT,                                  -- optional audit artifact
  metric_snapshot_before JSONB,                                 -- state at proposal time
  metric_snapshot_after  JSONB,                                 -- state after execution (for outcome scoring)
  outcome_verdict        outcome_verdict,                       -- did it actually help?
  outcome_measured_at    TIMESTAMP,
  created_at             TIMESTAMP NOT NULL DEFAULT now()
);

-- Behavioral learning: running approval stats per (account, target, action).
CREATE TABLE user_preferences (          -- founder_preferences
  id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  brand_id             UUID NOT NULL REFERENCES brands(id),     -- account_id
  platform             VARCHAR(50)  NOT NULL,                   -- target dimension
  action_type          VARCHAR(100) NOT NULL,
  approval_rate        REAL DEFAULT 0,                          -- approved / total
  avg_modification     REAL,                                    -- avg edit magnitude on modify
  auto_approve_eligible BOOLEAN DEFAULT FALSE,
  approved_count       INTEGER DEFAULT 0,
  rejected_count       INTEGER DEFAULT 0,
  modified_count       INTEGER DEFAULT 0,
  UNIQUE (brand_id, platform, action_type)
);

-- Optional: per-decision audit trail feeding richer learning context.
CREATE TABLE learning_log (
  id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  brand_id           UUID NOT NULL REFERENCES brands(id),
  recommendation_id  UUID REFERENCES recommendations(id),
  action_type        VARCHAR(100),
  platform           VARCHAR(50),
  analysis_type      VARCHAR(100),
  context            JSONB,
  outcome            VARCHAR(100),
  outcome_delta      REAL,
  outcome_verdict    outcome_verdict,
  founder_action     VARCHAR(50),       -- approved | rejected | modified  (user_action)
  modifier_details   JSONB,
  created_at         TIMESTAMP NOT NULL DEFAULT now()
);
```

> Built on Postgres + Drizzle ORM in the reference implementation. The `action_type`/`platform` enums are the only **domain-specific** part of the schema — replace those two enums and the table is yours.

---

## 8. Dependencies

**npm**

```jsonc
{
  "dependencies": {
    "@anthropic-ai/sdk": "^0.x",   // Claude analysis passes
    "drizzle-orm": "^0.x",         // typed DB access (any Drizzle-supported DB)
    "pg": "^8.x",                  // Postgres driver
    "next": "^14.x"                // API routes (any HTTP framework works)
  }
}
```

**Environment variables**

```bash
ANTHROPIC_API_KEY=sk-ant-...      # Claude API key (analyzer passes)
DATABASE_URL=postgres://user:pass@host:5432/db   # recommendations + learning store
```

The only hard external dependency is the **Claude API**. The DB layer is Drizzle/Postgres in the reference build but the pattern is storage-agnostic — anything that can hold the three tables works. The HTTP layer (Next.js route handlers) is interchangeable with Express/Fastify/Hono.

---

## 9. Porting Checklist

**Swap the domain prompts, keep the framework.** The pipeline (concurrent analyzers → scored recs → conflict dedup → human approval → learning loop) is the reusable asset; the ad-tech action types are just one instantiation.

- [ ] Replace `ANALYSIS_PROMPTS` with your domain's lenses (one prompt per "way of looking" at the data).
- [ ] Update `ALL_ANALYSIS_TYPES` to match — adding a lens is a one-line change.
- [ ] Rewrite the `SYSTEM_PROMPT` output contract for your `action_type` set; keep `confidence` / `risk` / `reasoning` / `projected_impact`.
- [ ] Redefine the `action_type` and `platform` (target-dimension) enums in the schema.
- [ ] Point `loadWindow()` / `loadDecisionHistory()` at your account's data store.
- [ ] Keep `analyzeAll`'s `Promise.all` fan-out with per-pass `.catch(() => [])` isolation.
- [ ] Keep `saveRecommendations`' conflict key `(target, actionType, status=pending)` — tune the comparison if needed.
- [ ] Wire the five endpoints (`list`, `approve`, `reject`, `modify`, `batch`); each MUST call `recordUserAction`.
- [ ] Keep `recordUserAction`'s counter math; decide your `autoApproveEligible` threshold on `approvalRate`.
- [ ] Feed `user_preferences` / `learning_log` back into the analyzer prompt as "historical outcomes."
- [ ] Set `ANTHROPIC_API_KEY` and `DATABASE_URL`.
- [ ] (Optional) Add outcome scoring: snapshot metrics before/after execution, write `outcome_verdict` to close the loop end to end.

---

*Built by [Quantana](https://quantana.in). MIT licensed.*
