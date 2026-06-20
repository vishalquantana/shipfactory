# Subscription Management & Recurring Billing

> A portable, provider-agnostic subscription lifecycle for SaaS apps: trial → active → past_due → cancelled, with HMAC webhook signature verification and access-gating that maps every subscription state to a concrete access level. Built on **Cashfree Subscriptions (India)** but designed so the payment provider is a single swappable layer — the lifecycle, access logic, and database schema are unchanged whether you run Cashfree, Stripe, or Razorpay.

---

## 1. Overview

This feature gives `<YourApp>` a complete recurring-billing lifecycle for a per-account paid plan. An account starts on a trial, converts to `active` after a successful first charge (with a recurring mandate), drops to `past_due` after repeated failed charges, and ends in `cancelled` either by user action or provider event.

The design has four layers, only one of which is provider-specific:

| Layer | Provider-agnostic? | File |
| --- | --- | --- |
| **Lifecycle state machine** | Yes | DB `status` enum + access logic |
| **Access gating** | Yes | `src/lib/subscription-access.ts` |
| **Database schema** | Yes | `src/lib/db/schema.ts` |
| **Provider client** | **No — swappable** | `src/lib/cashfree.ts` |
| **Webhook handler** | **No — swappable** (payload shape + signature scheme) | `src/app/api/webhooks/cashfree/route.ts` |

> **India-only note:** Cashfree Subscriptions uses UPI AutoPay / e-mandate, which is an India-specific rail. The provider client and the webhook payload/signature format are the **swappable layer**. To run elsewhere, replace `cashfree.ts` and the webhook parser with a Stripe or Razorpay equivalent — the lifecycle, access-gating, and DB design stay identical.

---

## 2. Subscription Lifecycle

### State diagram

```
                         createSubscription()
                                  │
                                  ▼
                          ┌───────────────┐
                          │     trial     │  trialStartsAt / trialEndsAt
                          └───────────────┘
                              │        │
        PAYMENT_SUCCESS       │        │  user cancels during trial
        (first charge +       │        │  (cancel endpoint, no provider call)
         mandate active)      │        │
                              ▼        ▼
                      ┌───────────┐  ┌───────────┐
                      │  active   │  │ cancelled │
                      └───────────┘  └───────────┘
                          │   ▲             ▲
   PAYMENT_FAILED ×3+     │   │ PAYMENT_    │
                          ▼   │ SUCCESS     │ SUBSCRIPTION_STATUS_CHANGE
                    ┌───────────┐ (retry)   │ == CANCELLED  /  cancel endpoint
                    │ past_due  │───────────┤
                    └───────────┘           │
                          │                 │
                          └─────────────────┘
```

### State → access mapping

Access is **derived from state at read time** (see §5), never stored as a flag:

| State | Access | Rule |
| --- | --- | --- |
| *(no subscription row)* | `FULL_ACCESS` | New/unmetered accounts are open by default |
| `trial` | `FULL_ACCESS` while `now < trialEndsAt`, else `SOFT_LOCK` | Trial window |
| `active` | `FULL_ACCESS` | Paid and current |
| `past_due` | `SOFT_LOCK` | Charge failed 3+ times |
| `cancelled` | `FULL_ACCESS` while `now < currentPeriodEnd`, else `SOFT_LOCK` | Honors already-paid period |

`SOFT_LOCK` means the app shell loads but gated actions are blocked behind an upgrade prompt — it is **not** a hard logout. This keeps the user able to re-subscribe.

---

## 3. Provider Client *(swappable layer — Cashfree → Stripe/Razorpay)*

`src/lib/cashfree.ts` is the only file that talks to the payment provider's REST API. It handles plan creation, subscription creation, the first authorization (mandate setup), recurring charges, cancellation, and status fetch.

Auth is via `x-client-id` / `x-client-secret` headers, and every mutating call carries an **idempotency key** so retries never double-charge.

```ts
const PLAN_ID = "<yourapp>_monthly_4999";
const PLAN_AMOUNT = 4999;

function getBaseUrl(): string {
  return process.env.CASHFREE_ENV === "production"
    ? "https://api.cashfree.com/pg"
    : "https://sandbox.cashfree.com/pg";
}

function getHeaders(idempotencyKey?: string): Record<string, string> {
  const headers: Record<string, string> = {
    "Content-Type": "application/json",
    "x-client-id": process.env.CASHFREE_APP_ID!,
    "x-client-secret": process.env.CASHFREE_SECRET_KEY!,
    "x-api-version": "2025-01-01",
  };
  if (idempotencyKey) {
    headers["x-idempotency-key"] = idempotencyKey;
  }
  return headers;
}

async function cfFetch<T>(path: string, options: RequestInit): Promise<T> {
  const url = `${getBaseUrl()}${path}`;
  const res = await fetch(url, options);
  if (!res.ok) {
    const body = await res.json().catch(() => ({}));
    const err = new Error(`Cashfree API error: ${res.status}`) as Error & {
      status: number; body: unknown;
    };
    err.status = res.status;
    err.body = body;
    throw err;
  }
  return res.json() as Promise<T>;
}

export const cashfree = {
  // Create the recurring plan once. 409 = already exists → treat as success (idempotent).
  async createPlan() {
    try {
      return await cfFetch<{ plan_id: string }>("/subscriptions/plans", {
        method: "POST",
        headers: getHeaders(),
        body: JSON.stringify({
          plan_id: PLAN_ID,
          plan_name: "<YourApp> Growth - Monthly",
          plan_type: "PERIODIC",
          plan_currency: "INR",
          plan_recurring_amount: PLAN_AMOUNT,
          plan_max_amount: PLAN_AMOUNT,
          plan_intervals: 1,
          plan_interval_type: "month",
          plan_max_cycles: 0,
        }),
      });
    } catch (err: unknown) {
      if (err && typeof err === "object" && "status" in err && err.status === 409) {
        return { plan_id: PLAN_ID };
      }
      throw err;
    }
  },

  // Create a subscription for an account's customer; returns provider subscription id.
  async createSubscription(params: {
    subscriptionId: string; customerName: string;
    customerEmail: string; customerPhone: string; returnUrl: string;
  }) {
    return cfFetch<{ subscription_id: string; cf_subscription_id: string }>(
      "/subscriptions", {
        method: "POST",
        headers: getHeaders(`subscribe_${params.subscriptionId}`),
        body: JSON.stringify({
          subscription_id: params.subscriptionId,
          customer_details: {
            customer_name: params.customerName,
            customer_email: params.customerEmail,
            customer_phone: params.customerPhone,
          },
          plan_details: { plan_id: PLAN_ID },
          subscription_meta: { return_url: params.returnUrl },
        }),
      });
  },

  // AUTH: set up the recurring mandate. Returns the hosted authorization_link (checkout URL).
  async raiseAuth(params: { subscriptionId: string; paymentId: string }) {
    return cfFetch<{
      cf_payment_id: string; payment_status: string;
      data: { authorization_link: string };
    }>("/subscriptions/payments", {
      method: "POST",
      headers: getHeaders(`auth_${params.paymentId}`),
      body: JSON.stringify({
        subscription_id: params.subscriptionId,
        payment_id: params.paymentId,
        payment_type: "AUTH",
      }),
    });
  },

  // CHARGE: pull a recurring payment against an active mandate.
  async raiseCharge(params: { subscriptionId: string; paymentId: string; amount: number }) {
    return cfFetch<{ cf_payment_id: string; payment_status: string }>(
      "/subscriptions/payments", {
        method: "POST",
        headers: getHeaders(`charge_${params.paymentId}`),
        body: JSON.stringify({
          subscription_id: params.subscriptionId,
          payment_id: params.paymentId,
          payment_type: "CHARGE",
          payment_amount: params.amount,
        }),
      });
  },

  async cancelSubscription(subscriptionId: string) {
    return cfFetch<{ subscription_status: string }>(
      `/subscriptions/${subscriptionId}/manage`, {
        method: "POST",
        headers: getHeaders(),
        body: JSON.stringify({ subscription_id: subscriptionId, action: "CANCEL" }),
      });
  },

  async fetchSubscription(subscriptionId: string) {
    return cfFetch<{
      subscription_id: string; subscription_status: string;
      authorization_details?: { payment_method?: string };
    }>(`/subscriptions/${subscriptionId}`, { method: "GET", headers: getHeaders() });
  },
};
```

**Provider-mapping cheatsheet** (when swapping):

| This client | Stripe equivalent | Razorpay equivalent |
| --- | --- | --- |
| `createPlan` | `prices.create` | `plans.create` |
| `createSubscription` | `subscriptions.create` (incomplete) | `subscriptions.create` |
| `raiseAuth` → `authorization_link` | `SetupIntent` / hosted Checkout `url` | `short_url` (auth txn) |
| `raiseCharge` | `invoices` auto-charge | `subscription.charge` |
| `cancelSubscription` | `subscriptions.cancel` | `subscriptions.cancel` |

---

## 4. Webhooks *(signature scheme is provider-specific; the pattern is universal)*

**This is the single most security-critical piece.** The webhook endpoint is publicly reachable, so the body **must** be authenticated before any DB write. The universal pattern:

1. Read the **raw** request body (do not parse first — HMAC is over the exact bytes).
2. Recompute the HMAC using your shared secret.
3. Compare against the provider-supplied signature header.
4. Reject (`401`) on mismatch, and **fail closed** (`500`) if the secret is not even configured.

Cashfree signs with `HMAC-SHA256(rawBody, secret)` → base64, in the `x-cashfree-signature` header. (Stripe uses `Stripe-Signature` with a timestamped scheme; Razorpay uses `X-Razorpay-Signature` hex — only this function changes.)

```ts
import { createHmac } from "crypto";

function verifySignature(rawBody: string, signature: string): boolean {
  const secret = process.env.CASHFREE_WEBHOOK_SECRET;
  if (!secret) return false;
  const computed = createHmac("sha256", secret).update(rawBody).digest("base64");
  return computed === signature;
}

export async function POST(req: NextRequest) {
  const rawBody = await req.text();                                  // raw bytes, not JSON
  const signature = req.headers.get("x-cashfree-signature") || "";

  // Fail CLOSED if the secret is missing — never accept unauthenticated webhooks.
  if (!process.env.CASHFREE_WEBHOOK_SECRET) {
    return NextResponse.json({ error: "Webhook not configured" }, { status: 500 });
  }
  if (!verifySignature(rawBody, signature)) {
    return NextResponse.json({ error: "Invalid signature" }, { status: 401 });
  }

  const payload = JSON.parse(rawBody);
  const eventType = payload.type as string;
  const data = payload.data as Record<string, unknown>;
  // ... look up local sub by provider subscription id, then handle event ...
}
```

### Event handling (state transitions)

The handler maps provider events onto the local lifecycle. Every payment write is **idempotent** (dedup by `cf_payment_id`).

| Event | Action |
| --- | --- |
| `PAYMENT_SUCCESS` / `PAYMENT_SUCCESS_SUBSCRIPTION` | Dedup on `cf_payment_id`; insert payment row (`success`); set sub `active`, set `currentPeriodStart`/`End` (+1 month), record payment method |
| `PAYMENT_FAILED` / `PAYMENT_FAILED_SUBSCRIPTION` | Insert payment row (`failed`, `onConflictDoNothing`); if `active` and **3+ failures**, move to `past_due` |
| `SUBSCRIPTION_STATUS_CHANGE` (== `CANCELLED`) | Set sub `cancelled`, stamp `cancelledAt` |
| `MANDATE_ACTIVE` | Set `hasRecurringMandate = true`, record payment method |

Key transition (success → active), real code:

```ts
case "PAYMENT_SUCCESS":
case "PAYMENT_SUCCESS_SUBSCRIPTION": {
  const payment = data.payment as Record<string, unknown>;
  const cfPaymentId = payment?.cf_payment_id as string;
  const amount = Math.round(((payment?.payment_amount as number) || 4999) * 100);

  // Idempotency: skip if we've already recorded this payment.
  if (cfPaymentId) {
    const existing = await db.query.subscriptionPayments.findFirst({
      where: eq(subscriptionPayments.cfPaymentId, cfPaymentId),
    });
    if (existing) return NextResponse.json({ ok: true });
  }

  await db.insert(subscriptionPayments).values({
    subscriptionId: sub.id,
    cfPaymentId: cfPaymentId || `unknown_${Date.now()}`,
    amountPaise: amount,
    status: "success",
    paymentMethod: (payment?.payment_method as string) || "unknown",
    paidAt: new Date(),
  });

  const periodStart = new Date();
  const periodEnd = new Date();
  periodEnd.setMonth(periodEnd.getMonth() + 1);

  await db.update(subscriptions).set({
    status: "active",
    paymentMethod: mapPaymentMethod(payment?.payment_method as string),
    currentPeriodStart: periodStart,
    currentPeriodEnd: periodEnd,
    updatedAt: new Date(),
  }).where(eq(subscriptions.id, sub.id));
  break;
}
```

Failure → past_due (only after a threshold, to absorb transient bank declines):

```ts
case "PAYMENT_FAILED":
case "PAYMENT_FAILED_SUBSCRIPTION": {
  // ... insert failed payment row (onConflictDoNothing) ...
  if (sub.status === "active") {
    const failedPayments = await db.query.subscriptionPayments.findMany({
      where: eq(subscriptionPayments.subscriptionId, sub.id),
    });
    const recentFailures = failedPayments.filter((p) => p.status === "failed").length;
    if (recentFailures >= 3) {
      await db.update(subscriptions)
        .set({ status: "past_due", updatedAt: new Date() })
        .where(eq(subscriptions.id, sub.id));
    }
  }
  break;
}
```

> **Unknown subscriptions return `200 OK`** (not an error) so the provider does not retry forever for rows you don't own.

---

## 5. Access Gating

`src/lib/subscription-access.ts` is the **single source of truth** for whether an account can use gated features. It computes access purely from current state + clock — no stored "is_locked" flag to drift out of sync.

```ts
import { db } from "./db";
import { subscriptions } from "./db/schema";
import { eq } from "drizzle-orm";

export enum AccessLevel {
  FULL_ACCESS = "FULL_ACCESS",
  SOFT_LOCK = "SOFT_LOCK",
}

export async function getAccessLevel(accountId: string): Promise<AccessLevel> {
  const sub = await db.query.subscriptions.findFirst({
    where: eq(subscriptions.brandId, accountId),
  });

  if (!sub) return AccessLevel.FULL_ACCESS;       // no row → open

  const now = new Date();

  if (sub.status === "trial") {
    return now < sub.trialEndsAt ? AccessLevel.FULL_ACCESS : AccessLevel.SOFT_LOCK;
  }
  if (sub.status === "active") {
    return AccessLevel.FULL_ACCESS;
  }
  if (sub.status === "past_due") {
    return AccessLevel.SOFT_LOCK;
  }
  if (sub.status === "cancelled") {
    // Honor the period the user already paid for.
    if (sub.currentPeriodEnd && now < sub.currentPeriodEnd) {
      return AccessLevel.FULL_ACCESS;
    }
    return AccessLevel.SOFT_LOCK;
  }
  return AccessLevel.SOFT_LOCK;                    // unknown state → fail closed
}
```

Design principles: **default-open** for accounts with no subscription, **fail-closed** for unknown states, and **honor paid time** after cancellation. The status endpoint (§6) inlines the same logic to return `accessLevel` to the client.

---

## 6. API Endpoints

All routes are guarded by auth (`requireOwnerAccess` for mutations, `requireBrandAccess` for reads) and scoped to `/accounts/[accountId]/...`.

### `POST .../subscribe` — start or resume a subscription

`src/app/api/brands/[brandId]/subscribe/route.ts`

1. Reject if an `active` subscription already exists (`400`).
2. Require the user's email (needed by the provider).
3. `cashfree.createPlan()` (idempotent), then `createSubscription()` if no provider id yet — and upsert the local row (`status: "trial"` for brand-new accounts).
4. `cashfree.raiseAuth()` to set up the mandate; return the hosted `authorization_link` as `checkoutUrl`.

```ts
await cashfree.createPlan();
const subscriptionId = `sub_${accountId}_${Date.now()}`;
const returnUrl = `${appUrl}/account/${accountId}/subscription?status={status}`;

let cfSubId = existing?.cfSubscriptionId;
if (!cfSubId) {
  const cfSub = await cashfree.createSubscription({ subscriptionId, customerName, customerEmail: user.email, customerPhone: "9999999999", returnUrl });
  cfSubId = cfSub.subscription_id;
  // upsert local row: existing → patch cfSubscriptionId; new → insert status "trial"
}

const authResult = await cashfree.raiseAuth({ subscriptionId: cfSubId, paymentId: `auth_${accountId}_${Date.now()}` });
const checkoutUrl = authResult.data?.authorization_link;       // 502 if missing
return NextResponse.json({ checkoutUrl, subscriptionId: cfSubId });
```

### `GET .../subscription` — status + access level

`src/app/api/brands/[brandId]/subscription/route.ts`

Returns the computed `accessLevel` plus the subscription summary (`status`, `paymentMethod`, `hasRecurringMandate`, trial/period timestamps, `cancelledAt`). Returns `{ accessLevel: "FULL_ACCESS", subscription: null }` when no row exists.

```ts
const accessLevel = !sub
  ? "FULL_ACCESS"
  : sub.status === "active" ? "FULL_ACCESS"
  : sub.status === "trial" && new Date() < sub.trialEndsAt ? "FULL_ACCESS"
  : sub.status === "cancelled" && sub.currentPeriodEnd && new Date() < sub.currentPeriodEnd ? "FULL_ACCESS"
  : "SOFT_LOCK";
```

### `POST .../subscription/cancel` — cancel

`src/app/api/brands/[brandId]/subscription/cancel/route.ts`

- `404` if no subscription; `400` if already `cancelled`.
- **Trial cancel** is local-only (no provider call): set `cancelled` + `cancelledAt`.
- Otherwise call `cashfree.cancelSubscription()` (`502` on provider failure), then mark `cancelled` locally and return `accessUntil: currentPeriodEnd` so the UI can show remaining paid time.

---

## 7. Database Schema

Provider-agnostic. The only provider-coupled columns are `cf_subscription_id` / `cf_plan_id` / `cf_payment_id` — rename to `provider_*` if you prefer. Drizzle (`pgEnum` / `pgTable`) maps to this Postgres DDL:

```sql
-- ─── Enums ────────────────────────────────────────────────────────────────
CREATE TYPE subscription_status AS ENUM ('trial', 'active', 'past_due', 'cancelled');
CREATE TYPE subscription_payment_method AS ENUM ('upi_autopay', 'card', 'netbanking', 'wallet', 'unknown');
CREATE TYPE subscription_payment_status AS ENUM ('success', 'failed', 'pending');

-- ─── subscriptions (one per account) ──────────────────────────────────────
CREATE TABLE subscriptions (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  brand_id              UUID NOT NULL REFERENCES brands(id),   -- = account_id
  cf_subscription_id    VARCHAR(255),                          -- provider sub id
  cf_plan_id            VARCHAR(100),                          -- provider plan id
  status                subscription_status NOT NULL DEFAULT 'trial',
  payment_method        subscription_payment_method DEFAULT 'unknown',
  has_recurring_mandate BOOLEAN DEFAULT false,
  trial_starts_at       TIMESTAMP NOT NULL,
  trial_ends_at         TIMESTAMP NOT NULL,
  current_period_start  TIMESTAMP,
  current_period_end    TIMESTAMP,
  cancelled_at          TIMESTAMP,
  created_at            TIMESTAMP NOT NULL DEFAULT now(),
  updated_at            TIMESTAMP NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX subscriptions_brand_id_unique ON subscriptions (brand_id);

-- ─── subscription_payments (ledger, one row per charge attempt) ────────────
CREATE TABLE subscription_payments (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  subscription_id UUID NOT NULL REFERENCES subscriptions(id),
  cf_payment_id   VARCHAR(255),                                -- provider payment id (dedup key)
  amount_paise    INTEGER NOT NULL,                            -- minor units (₹49.99 → 4999_paise... stored ×100)
  status          subscription_payment_status NOT NULL DEFAULT 'pending',
  payment_method  VARCHAR(50),
  paid_at         TIMESTAMP,
  created_at      TIMESTAMP NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX subscription_payments_cf_id_unique ON subscription_payments (cf_payment_id);
```

Notes:
- **One subscription per account** enforced by the unique index on `brand_id`.
- The **unique index on `cf_payment_id`** is what makes webhook ingestion idempotent (`onConflictDoNothing`).
- Amounts are stored in **paise** (minor units, integer) to avoid float rounding.

---

## 8. Dependencies

### npm

```json
{
  "drizzle-orm": "^0.45.1",   // query builder / schema (swappable for Prisma/raw SQL)
  "pg": "^8.20.0",            // Postgres driver
  "next": "16.1.6"            // App Router route handlers
}
```

No payment SDK is required — the provider client uses native `fetch` and Node's built-in `crypto` (`createHmac`) for signature verification.

### Environment variables

| Var | Purpose |
| --- | --- |
| `CASHFREE_ENV` | `production` or sandbox (selects API base URL) |
| `CASHFREE_APP_ID` | Provider client id (`x-client-id`) |
| `CASHFREE_SECRET_KEY` | Provider client secret (`x-client-secret`) |
| `CASHFREE_WEBHOOK_SECRET` | HMAC secret for webhook signature verify — **required; fail closed if absent** |
| `NEXT_PUBLIC_APP_URL` | Base URL for building the post-checkout `return_url` |
| `DATABASE_URL` | Postgres connection string |

> When swapping providers, replace the `CASHFREE_*` vars with `STRIPE_*` / `RAZORPAY_*` equivalents (API key, webhook signing secret).

---

## 9. Porting Checklist

- [ ] Create the three Postgres enums and the `subscriptions` + `subscription_payments` tables (§7), including both unique indexes.
- [ ] Copy `subscription-access.ts` — the `AccessLevel` enum and `getAccessLevel()` are **provider-agnostic; use as-is**. Rename `brandId` → your account key.
- [ ] Wire `getAccessLevel()` into your auth/middleware so gated routes check `SOFT_LOCK`.
- [ ] Add the three API endpoints (subscribe / status / cancel), adapting the auth guards to your stack.
- [ ] Set the env vars (§8); ensure `*_WEBHOOK_SECRET` is set in every environment (handler fails closed without it).
- [ ] Register the webhook URL with the provider; confirm the **raw-body** signature check passes end-to-end with a test event.
- [ ] Verify idempotency: re-deliver the same `PAYMENT_SUCCESS` and confirm no duplicate ledger row and no double-activation.
- [ ] **Swap provider (Stripe / Razorpay):** rewrite only `cashfree.ts` (map plan/subscription/auth/charge/cancel per §3 cheatsheet) and the webhook **parser + `verifySignature`** (Stripe `Stripe-Signature` timestamped HMAC; Razorpay `X-Razorpay-Signature` hex HMAC). Keep the event→state transition table (§4), the access logic (§5), and the schema (§7) unchanged.
- [ ] Decide trial length: this build sets `trialEndsAt = now` on insert (effectively no trial / charge-first). Adjust the insert in `subscribe` to grant a real trial window.

---

*Built by [Quantana](https://quantana.in). MIT licensed.*
