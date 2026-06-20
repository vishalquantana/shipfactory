# Waitlist / Email Capture

> A drop-in waitlist endpoint for collecting early-access signups in any Next.js SaaS. It captures an email plus UTM attribution, blocks bots with a honeypot field and a submission-timing check, throttles abusive IPs, dedupes against existing rows, persists to Postgres, and fires a team notification email — all without a third-party form service.

## 1. Overview

This feature ships a complete email-capture flow for `<YourApp>`. Capabilities:

- **Email capture** — validates and stores a signup email, returning the new entry's waitlist position.
- **Bot protection** — a hidden honeypot field plus a minimum form-fill time (1.5s) silently absorb automated submissions without storing them.
- **IP rate limiting** — caps each IP at 5 requests/minute to stop scripted abuse.
- **Dedupe** — rejects emails already on the list (also enforced by a `UNIQUE` constraint at the DB level).
- **UTM tracking** — captures `utm_source`, `utm_medium`, `utm_campaign`, and the landing `source` path for attribution.
- **Team notification** — fires a fire-and-forget email to your team via [Resend](https://resend.com) on each real signup (optional — skipped if no API key).

## 2. Flow

```
  Visitor                Frontend Form              POST /api/waitlist                 Postgres
    |                         |                            |                              |
    |  type email + submit    |                            |                              |
    |------------------------>|                            |                              |
    |                         |  { email, source, utm_*,   |                              |
    |                         |    _hp_check, _t }         |                              |
    |                         |--------------------------->|                              |
    |                         |                            |-- rate limit by IP? --429    |
    |                         |                            |-- honeypot filled? --> fake OK (no store)
    |                         |                            |-- submitted < 1.5s? --> fake OK (no store)
    |                         |                            |-- invalid email? -----> 400  |
    |                         |                            |-- SELECT WHERE email --------->|
    |                         |                            |<-- exists? ------------- 409   |
    |                         |                            |-- INSERT new row ------------->|
    |                         |                            |-- SELECT COUNT(*) ----------->|
    |                         |                            |<-- position ------------------|
    |                         |                            |-- sendNotification() ~~> Resend (async)
    |                         |<-- { message, position } --|                              |
    |<-- "We'll be in touch" -|                            |                              |
```

## 3. Frontend Form

A client component. Two anti-bot mechanisms live here: an off-screen honeypot `<input name="_hp_check">` that real users never see (and bots auto-fill), and a `_t` timestamp captured on mount so the backend can reject instant submissions. UTM params are read from the URL at submit time.

```tsx
"use client";

import { useState, useRef, useEffect } from "react";

interface WaitlistFormProps {
  className?: string;
}

export default function WaitlistForm({ className = "" }: WaitlistFormProps) {
  const [email, setEmail] = useState("");
  const [status, setStatus] = useState<"idle" | "loading" | "success" | "error">("idle");
  const [message, setMessage] = useState("");
  const loadTimeRef = useRef<number>(Date.now());

  useEffect(() => {
    loadTimeRef.current = Date.now();
  }, []);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setStatus("loading");

    const form = e.target as HTMLFormElement;
    const honeypot = (form.elements.namedItem("_hp_check") as HTMLInputElement)?.value;

    try {
      const res = await fetch("/api/waitlist", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          email,
          source: window.location.pathname,
          utm_source: new URLSearchParams(window.location.search).get("utm_source"),
          utm_medium: new URLSearchParams(window.location.search).get("utm_medium"),
          utm_campaign: new URLSearchParams(window.location.search).get("utm_campaign"),
          _hp_check: honeypot,
          _t: loadTimeRef.current,
        }),
      });

      const data = await res.json();

      if (res.ok) {
        setStatus("success");
        setMessage("We'll get in touch with you shortly");
        setEmail("");
      } else {
        setStatus("error");
        setMessage(data.error || "Something went wrong. Try again.");
      }
    } catch {
      setStatus("error");
      setMessage("Something went wrong. Try again.");
    }
  }

  if (status === "success") {
    return (
      <div className={`text-center ${className}`}>
        <p className="text-green-600 font-semibold">{message}</p>
      </div>
    );
  }

  return (
    <>
      <form onSubmit={handleSubmit} className={`flex gap-3 max-w-md mx-auto ${className}`}>
        {/* Honeypot — invisible to real users, bots auto-fill it */}
        <input
          type="text"
          name="_hp_check"
          tabIndex={-1}
          autoComplete="off"
          aria-hidden="true"
          style={{ position: "absolute", left: "-9999px", opacity: 0, height: 0, width: 0 }}
        />
        <label htmlFor="waitlist-email" className="sr-only">Email address</label>
        <input
          id="waitlist-email"
          type="email"
          required
          placeholder="you@yourbrand.com"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          aria-label="Email address"
          className="flex-1 rounded-lg border px-4 py-3.5 text-sm outline-none transition-colors"
        />
        <button
          type="submit"
          disabled={status === "loading"}
          aria-busy={status === "loading"}
          className="whitespace-nowrap rounded-lg px-6 py-3.5 text-sm font-semibold text-white transition-colors disabled:opacity-50"
        >
          {status === "loading" ? "Signing up..." : "Get Started for FREE"}
        </button>
      </form>
      <div aria-live="polite" aria-atomic="true">
        {status === "error" && (
          <p className="mt-2 text-center text-xs text-red-500">{message}</p>
        )}
      </div>
    </>
  );
}
```

## 4. Backend Endpoint

`app/api/waitlist/route.ts`. Order of operations: IP rate limit → honeypot → timing → email validation → dedupe → insert → position lookup → async notification. The two bot checks return a **fake success** so bots believe they succeeded and don't retry.

```ts
import { NextRequest, NextResponse } from "next/server";
import { query } from "@/lib/db";

const NOTIFY_RECIPIENTS = [
  "founder@<yourapp>.com",
  "team@<yourapp>.com",
];

async function sendNotification(params: {
  email: string;
  source: string | null;
  utm_source: string | null;
  utm_medium: string | null;
  utm_campaign: string | null;
  position: number | string;
}) {
  const apiKey = process.env.RESEND_API_KEY;
  if (!apiKey) {
    console.warn("RESEND_API_KEY missing — skipping waitlist notification email");
    return;
  }

  const { email, source, utm_source, utm_medium, utm_campaign, position } = params;
  const rows = [
    ["Email", email],
    ["Position", String(position)],
    ["Source", source || "—"],
    ["UTM Source", utm_source || "—"],
    ["UTM Medium", utm_medium || "—"],
    ["UTM Campaign", utm_campaign || "—"],
  ];

  const html = `
    <div style="font-family: Arial, sans-serif; color: #1a1714;">
      <h2 style="color: #dc2626; margin: 0 0 16px;">New <YourApp> waitlist signup</h2>
      <table style="border-collapse: collapse; font-size: 14px;">
        ${rows
          .map(
            ([k, v]) =>
              `<tr><td style="padding: 6px 12px 6px 0; color: #6b6358;"><strong>${k}</strong></td><td style="padding: 6px 0;">${v}</td></tr>`
          )
          .join("")}
      </table>
    </div>
  `;

  try {
    const res = await fetch("https://api.resend.com/emails", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        from: "<YourApp> <noreply@<yourapp>.com>",
        to: NOTIFY_RECIPIENTS,
        subject: `New waitlist signup: ${email}`,
        html,
      }),
    });
    if (!res.ok) {
      const text = await res.text();
      console.error("Resend notification failed:", res.status, text);
    }
  } catch (err) {
    console.error("Resend notification error:", err);
  }
}

// In-memory rate limiter (per IP, resets on cold start — good enough for serverless)
const rateLimitMap = new Map<string, { count: number; resetAt: number }>();
const RATE_LIMIT_WINDOW = 60_000; // 1 minute
const RATE_LIMIT_MAX = 5; // 5 requests per minute per IP

function isRateLimited(ip: string): boolean {
  const now = Date.now();
  const entry = rateLimitMap.get(ip);

  if (!entry || now > entry.resetAt) {
    rateLimitMap.set(ip, { count: 1, resetAt: now + RATE_LIMIT_WINDOW });
    return false;
  }

  entry.count++;
  return entry.count > RATE_LIMIT_MAX;
}

export async function POST(request: NextRequest) {
  // Rate limiting by IP
  const ip = request.headers.get("x-forwarded-for")?.split(",")[0]?.trim() ||
    request.headers.get("x-real-ip") || "unknown";

  if (isRateLimited(ip)) {
    return NextResponse.json(
      { error: "Too many requests. Please try again in a minute." },
      { status: 429 }
    );
  }

  try {
    const body = await request.json();
    const { email, source, utm_source, utm_medium, utm_campaign, _hp_check, _t } = body;

    // Honeypot: if the hidden field has a value, it's a bot
    if (_hp_check) {
      // Silently accept but don't store — bots think they succeeded
      return NextResponse.json({
        message: "You're on the waitlist! We'll be in touch soon.",
        position: 1,
      });
    }

    // Timing check: reject if submitted faster than 1.5 seconds (bots fill instantly)
    if (_t && Date.now() - Number(_t) < 1500) {
      return NextResponse.json({
        message: "You're on the waitlist! We'll be in touch soon.",
        position: 1,
      });
    }

    // Validate email
    if (!email || !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
      return NextResponse.json(
        { error: "Please enter a valid email address." },
        { status: 400 }
      );
    }

    // Check for duplicate
    const existing = await query("SELECT id FROM waitlist WHERE email = $1", [email]);
    if (existing.rows.length > 0) {
      return NextResponse.json(
        { error: "You're already on the waitlist!" },
        { status: 409 }
      );
    }

    // Insert
    await query(
      "INSERT INTO waitlist (email, source, utm_source, utm_medium, utm_campaign) VALUES ($1, $2, $3, $4, $5)",
      [email, source || null, utm_source || null, utm_medium || null, utm_campaign || null]
    );

    // Get position
    const countResult = await query("SELECT COUNT(*) as count FROM waitlist");
    const position = countResult.rows[0].count;

    // Fire-and-forget notification to team
    sendNotification({
      email,
      source: source || null,
      utm_source: utm_source || null,
      utm_medium: utm_medium || null,
      utm_campaign: utm_campaign || null,
      position,
    }).catch((err) => console.error("Notification dispatch failed:", err));

    return NextResponse.json({
      message: `You're #${position} on the waitlist! We'll be in touch soon.`,
      position,
    });
  } catch (error) {
    console.error("Waitlist error:", error);
    return NextResponse.json(
      { error: "Something went wrong. Please try again." },
      { status: 500 }
    );
  }
}
```

> **Rate limiter caveat:** The limiter is a process-local `Map`. It **resets on every cold start / restart** and is **not shared across instances** — so on a multi-instance or autoscaled deployment, the effective limit is `RATE_LIMIT_MAX × instance_count`, and a restart wipes all counters. It's fine as a cheap first line of defense on a single instance. For real multi-instance enforcement, back it with Redis (e.g. `@upstash/ratelimit` or a Redis `INCR` + `EXPIRE` keyed on IP). The honeypot, timing check, and the DB `UNIQUE` constraint do not depend on this limiter.

## 5. Database Schema

Postgres. The `UNIQUE` constraint on `email` is the durable backstop for dedupe (the in-app `SELECT` check is a friendlier fast path).

```sql
CREATE TABLE IF NOT EXISTS waitlist (
  id            SERIAL PRIMARY KEY,
  email         TEXT UNIQUE NOT NULL,
  source        TEXT,          -- landing path, e.g. "/pricing"
  utm_source    TEXT,
  utm_medium    TEXT,
  utm_campaign  TEXT,
  created_at    TIMESTAMP DEFAULT NOW()
);
```

The reference implementation runs this via a tiny migration script (`lib/migrate.ts`) using a shared `pg` `Pool`:

```ts
import { getPool } from "./db";

async function migrate() {
  const pool = getPool();
  await pool.query(`
    CREATE TABLE IF NOT EXISTS waitlist (
      id SERIAL PRIMARY KEY,
      email TEXT UNIQUE NOT NULL,
      source TEXT,
      utm_source TEXT,
      utm_medium TEXT,
      utm_campaign TEXT,
      created_at TIMESTAMP DEFAULT NOW()
    )
  `);
  console.log("Migration complete: waitlist table created.");
  await pool.end();
}

migrate().catch(console.error);
```

The `query` helper (`lib/db.ts`) lazily initializes a connection pool from `DATABASE_URL`:

```ts
import { Pool } from "pg";

let pool: Pool | null = null;

export function getPool(): Pool {
  if (!pool) {
    if (!process.env.DATABASE_URL) {
      throw new Error("DATABASE_URL is not set");
    }
    pool = new Pool({ connectionString: process.env.DATABASE_URL, max: 5 });
  }
  return pool;
}

export async function query(text: string, params?: unknown[]) {
  const client = await getPool().connect();
  try {
    return await client.query(text, params);
  } finally {
    client.release();
  }
}
```

## 6. Dependencies

**npm packages**

```bash
npm install pg
npm install -D @types/pg
# Next.js + React are assumed (App Router). No SDK needed for Resend —
# the notification uses a plain fetch() to the Resend REST API.
```

**Environment variables**

| Variable          | Required | Purpose                                                                 |
| ----------------- | -------- | ----------------------------------------------------------------------- |
| `DATABASE_URL`    | **Yes**  | Postgres connection string. The endpoint and migration both fail without it. |
| `RESEND_API_KEY`  | Optional | Enables team notification emails. If unset, signups still work — the notification is skipped with a console warning. |

## 7. Porting Checklist

- [ ] Install `pg` and `@types/pg`; confirm Next.js App Router is in place.
- [ ] Copy `lib/db.ts` (pool + `query` helper) and `lib/migrate.ts`.
- [ ] Set `DATABASE_URL` and run the migration to create the `waitlist` table.
- [ ] Copy `app/api/waitlist/route.ts`; replace `<YourApp>` branding and `NOTIFY_RECIPIENTS`.
- [ ] (Optional) Set `RESEND_API_KEY` and update the verified `from:` sender domain.
- [ ] Copy the `WaitlistForm` client component; restyle to your design system.
- [ ] Verify the honeypot input renders off-screen and is not focusable (`tabIndex={-1}`).
- [ ] Test the timing gate: a sub-1.5s submission should return fake success and store nothing.
- [ ] Test rate limiting: the 6th request from one IP within 60s returns `429`.
- [ ] Test dedupe: re-submitting an existing email returns `409`.
- [ ] Confirm UTM params propagate from the URL through to the stored row.
- [ ] For multi-instance/autoscaled deploys, replace the in-memory limiter with Redis.

---

*Built by [Quantana](https://quantana.in). MIT licensed.*
