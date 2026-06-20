# Passwordless OTP Email Authentication

> A passwordless sign-in flow where users authenticate with a 6-digit one-time code emailed to them — no passwords to store, reset, or leak. Users enter their email, receive a time-limited code, and verify it to create a session. Built on Next.js App Router, NextAuth (Credentials provider), Drizzle ORM + Postgres, and Resend for email delivery.

## 1. Overview

This feature replaces password authentication with a single-factor email OTP flow. Core capabilities:

- **Two-step login UI** — step 1 collects the email, step 2 collects the 6-digit code.
- **6-digit numeric OTP** generated server-side, stored hashed-or-plain in Postgres with a 10-minute expiry.
- **Rate limiting** — a maximum of 3 codes per email per rolling hour (server-enforced) plus a 60-second client-side resend cooldown.
- **Single-use codes** — each code is marked `used` once redeemed and cannot be replayed.
- **Auto user provisioning** — first successful verification creates the user record (`authProvider = "otp"`).
- **Session creation** via NextAuth Credentials provider (JWT session strategy).
- **Test/QA bypass email** — one allow-listed address accepts any well-formed 6-digit code without sending mail, so automated E2E tests and reviewers can log in without an inbox.
- **Swappable email provider** — Resend is isolated behind a single `sendOtpEmail()` function.

The feature touches five surfaces: a login page, two API routes, an OTP service module, an email module, and one database table.

## 2. User Flow

```
┌──────────────┐
│  /login      │  Step 1: email
│  enter email │
└──────┬───────┘
       │ POST /api/auth/send-otp { email }
       ▼
┌──────────────────────────────────────────────┐
│ send-otp route                               │
│  • normalize email (lowercase + trim)        │
│  • if bypass email → return success (no mail) │
│  • checkRateLimit (≤3 codes / hour)           │
│      └─ over limit → 429                       │
│  • createOtp → insert row (code, expiresAt)   │
│  • sendOtpEmail via Resend                    │
│      └─ send fails → 500                       │
│  • return { success: true }                   │
└──────┬───────────────────────────────────────┘
       │ 200 → UI advances to step 2, starts 60s resend cooldown
       ▼
┌──────────────┐
│  /login      │  Step 2: enter 6-digit code
│  enter code  │  (Resend / Change email available)
└──────┬───────┘
       │ signIn("credentials", { email, code })
       ▼
┌──────────────────────────────────────────────┐
│ NextAuth authorize()  → verifyOtp(email,code) │
│  • bypass email + 6 digits → valid            │
│  • else: find unused, unexpired matching row  │
│      └─ none → return null (401, invalid)     │
│  • mark row used = true                        │
│  • find-or-create user (authProvider="otp")    │
│  • return user → JWT session issued            │
└──────┬───────────────────────────────────────┘
       │ session cookie set
       ▼
   redirect to /  (authenticated)
```

> Note: the standalone `POST /api/auth/verify-otp` route exists for non-NextAuth clients, but the web UI verifies *through* NextAuth's `authorize()` callback so that a session is created atomically with verification.

## 3. Frontend

A single client component (`src/app/(auth)/login/page.tsx`) drives both steps via a `step` state machine (`"email" | "otp"`). It uses `next-auth/react` for the session and sign-in.

### State & redirect-if-authenticated

```tsx
"use client";

import { signIn, useSession } from "next-auth/react";
import { useState, useEffect } from "react";
import { useRouter } from "next/navigation";

export default function LoginPage() {
  const { status } = useSession();
  const [email, setEmail] = useState("");
  const [code, setCode] = useState("");
  const [step, setStep] = useState<"email" | "otp">("email");
  const [error, setError] = useState("");
  const [loading, setLoading] = useState(false);
  const [cooldown, setCooldown] = useState(0);
  const router = useRouter();

  // Already signed in → bounce to app
  useEffect(() => {
    if (status === "authenticated") router.replace("/");
  }, [status, router]);

  // Prefill last-used email from localStorage
  useEffect(() => {
    const saved = localStorage.getItem("<yourapp>_login_email");
    if (saved) setEmail(saved);
  }, []);
```

### Step 1 — request the code

```tsx
  async function handleSendOtp(e: React.FormEvent) {
    e.preventDefault();
    setError("");
    setLoading(true);

    const res = await fetch("/api/auth/send-otp", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ email }),
    });

    setLoading(false);

    if (!res.ok) {
      const data = await res.json();
      setError(data.error || "Failed to send code");
      return;
    }

    localStorage.setItem("<yourapp>_login_email", email);
    setStep("otp");
    // Start 60s resend cooldown
    setCooldown(60);
    const timer = setInterval(() => {
      setCooldown((c) => {
        if (c <= 1) { clearInterval(timer); return 0; }
        return c - 1;
      });
    }, 1000);
  }
```

### Step 2 — verify through NextAuth

The verify step does **not** call `/api/auth/verify-otp` directly; it calls NextAuth `signIn` so that verification and session creation happen together.

```tsx
  async function handleVerifyOtp(e: React.FormEvent) {
    e.preventDefault();
    setError("");
    setLoading(true);

    // authorize() in the NextAuth config performs the OTP check
    const result = await signIn("credentials", {
      email,
      code,
      redirect: false,
    });

    setLoading(false);

    if (result?.ok) {
      router.push("/");
    } else {
      setError("Invalid or expired code. Please try again.");
    }
  }
```

### Inputs & validation

- **Email input**: `type="email"`, `required`, `autoFocus`. The browser enforces basic email format; the server re-normalizes (`toLowerCase().trim()`).
- **Code input**: numeric only — strips non-digits on change, limits to 6 chars, and disables submit until exactly 6 digits:

```tsx
<Input
  id="code"
  type="text"
  inputMode="numeric"
  pattern="[0-9]{6}"
  maxLength={6}
  value={code}
  onChange={(e) => setCode(e.target.value.replace(/\D/g, ""))}
  placeholder="000000"
  required
  autoFocus
  className="text-center text-2xl tracking-widest"
/>
...
<Button type="submit" className="w-full" disabled={loading || code.length !== 6}>
  {loading ? "Verifying..." : "Verify & sign in"}
</Button>
```

### Secondary actions (step 2)

- **Change email** — resets `step` to `"email"`, clears the code and error.
- **Resend code** — re-invokes `handleSendOtp`, disabled while `cooldown > 0`, labelled `Resend in {cooldown}s`.

### UI states summary

| State | Trigger | Behavior |
|-------|---------|----------|
| `loading` | network in flight | buttons disabled, label switches to "Sending…/Verifying…" |
| `error` | non-2xx response | red helper text under the form |
| `cooldown` | after a successful send | resend disabled with live countdown |
| `status === "loading" \| "authenticated"` | session check | renders a "Loading…" splash instead of the form |

## 4. Backend API

Two route handlers plus a shared OTP service module (`src/lib/otp.ts`).

### OTP service — `src/lib/otp.ts`

Constants and helpers. Note `BYPASS_EMAIL` is the QA/test allow-list address.

```ts
import { db } from "./db";
import { otpCodes } from "./db/schema";
import { eq, and, gt, gte, desc } from "drizzle-orm";

const BYPASS_EMAIL = "<qa-bypass@yourapp.com>";   // test-only allow-list
const OTP_EXPIRY_MINUTES = 10;
const MAX_OTPS_PER_HOUR = 3;

export function generateOtp(): string {
  return Math.floor(100000 + Math.random() * 900000).toString();
}

export function isBypassEmail(email: string): boolean {
  return email.toLowerCase() === BYPASS_EMAIL;
}

export async function checkRateLimit(email: string): Promise<boolean> {
  const oneHourAgo = new Date(Date.now() - 60 * 60 * 1000);
  const recentCodes = await db
    .select()
    .from(otpCodes)
    .where(
      and(
        eq(otpCodes.email, email.toLowerCase()),
        gte(otpCodes.createdAt, oneHourAgo)
      )
    );
  return recentCodes.length < MAX_OTPS_PER_HOUR;
}

export async function createOtp(email: string): Promise<string> {
  const code = generateOtp();
  const expiresAt = new Date(Date.now() + OTP_EXPIRY_MINUTES * 60 * 1000);
  await db.insert(otpCodes).values({
    email: email.toLowerCase(),
    code,
    expiresAt,
  });
  return code;
}

export async function verifyOtp(email: string, code: string): Promise<boolean> {
  // Test bypass: any well-formed 6-digit code succeeds for the allow-listed email
  if (isBypassEmail(email) && code.length === 6 && /^\d{6}$/.test(code)) {
    return true;
  }

  const now = new Date();
  const [otp] = await db
    .select()
    .from(otpCodes)
    .where(
      and(
        eq(otpCodes.email, email.toLowerCase()),
        eq(otpCodes.code, code),
        eq(otpCodes.used, false),
        gt(otpCodes.expiresAt, now)        // not expired
      )
    )
    .orderBy(desc(otpCodes.createdAt))      // newest first
    .limit(1);

  if (!otp) return false;

  // Single-use: burn the code so it can't be replayed
  await db.update(otpCodes).set({ used: true }).where(eq(otpCodes.id, otp.id));
  return true;
}
```

**Security notes for porting:**
- This reference stores the code in **plaintext**. For production, hash the code (e.g. `sha256(code + email)`) before insert and compare the hash on verify — change the `code` column accordingly and never log raw codes.
- `generateOtp()` uses `Math.random()`. For a hardened implementation use a CSPRNG (`crypto.randomInt(100000, 1000000)`).
- Rate limiting counts rows in the last hour; pair it with deletion/expiry cleanup so the table doesn't grow unbounded.

### `POST /api/auth/send-otp` — `src/app/api/auth/send-otp/route.ts`

```ts
import { NextResponse } from "next/server";
import { checkRateLimit, createOtp, isBypassEmail } from "@/lib/otp";
import { sendOtpEmail } from "@/lib/email";

export async function POST(req: Request) {
  const { email } = await req.json();

  if (!email || typeof email !== "string") {
    return NextResponse.json({ error: "Email is required" }, { status: 400 });
  }

  const normalizedEmail = email.toLowerCase().trim();

  // Bypass email skips rate limit, storage, and email send entirely
  if (isBypassEmail(normalizedEmail)) {
    return NextResponse.json({ success: true });
  }

  // Rate limit: max 3 per hour
  const allowed = await checkRateLimit(normalizedEmail);
  if (!allowed) {
    return NextResponse.json(
      { error: "Too many requests. Try again later." },
      { status: 429 }
    );
  }

  const code = await createOtp(normalizedEmail);

  try {
    await sendOtpEmail(normalizedEmail, code);
  } catch {
    return NextResponse.json(
      { error: "Failed to send email. Try again." },
      { status: 500 }
    );
  }

  return NextResponse.json({ success: true });
}
```

| Status | Meaning |
|--------|---------|
| 200 `{ success: true }` | code sent (or bypass email) |
| 400 | missing/invalid email |
| 429 | rate limit exceeded |
| 500 | email provider failed |

### `POST /api/auth/verify-otp` — `src/app/api/auth/verify-otp/route.ts`

Standalone verification for non-NextAuth clients (mobile, API consumers). The web UI uses NextAuth instead.

```ts
import { NextResponse } from "next/server";
import { verifyOtp } from "@/lib/otp";

export async function POST(req: Request) {
  const { email, code } = await req.json();

  if (!email || !code) {
    return NextResponse.json({ error: "Email and code are required" }, { status: 400 });
  }

  const valid = await verifyOtp(email.toLowerCase().trim(), code.trim());

  if (!valid) {
    return NextResponse.json({ error: "Invalid or expired code" }, { status: 401 });
  }

  return NextResponse.json({ verified: true });
}
```

## 5. Database Schema

One table, `otp_codes`, with an index on `email` (used by both the rate-limit count and the verify lookup).

### SQL (Postgres)

```sql
CREATE TABLE "otp_codes" (
  "id"         uuid PRIMARY KEY DEFAULT gen_random_uuid() NOT NULL,
  "email"      varchar(255) NOT NULL,
  "code"       varchar(6) NOT NULL,
  "expires_at" timestamp NOT NULL,
  "used"       boolean DEFAULT false,
  "created_at" timestamp DEFAULT now() NOT NULL
);

CREATE INDEX "idx_otp_codes_email" ON "otp_codes" USING btree ("email");
```

### Drizzle definition (`src/lib/db/schema.ts`)

```ts
export const otpCodes = pgTable("otp_codes", {
  id: uuid("id").primaryKey().defaultRandom(),
  email: varchar("email", { length: 255 }).notNull(),
  code: varchar("code", { length: 6 }).notNull(),
  expiresAt: timestamp("expires_at").notNull(),
  used: boolean("used").default(false),
  createdAt: timestamp("created_at").defaultNow().notNull(),
}, (table) => [
  index("idx_otp_codes_email").on(table.email),
]);
```

### Supporting `users` table (for session/provisioning)

```ts
export const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  email: varchar("email", { length: 255 }).notNull().unique(),
  name: varchar("name", { length: 255 }),
  image: text("image"),
  authProvider: varchar("auth_provider", { length: 50 }), // "otp" for this flow
  createdAt: timestamp("created_at").defaultNow().notNull(),
});
```

> Recommended addition (not in the reference): a scheduled cleanup `DELETE FROM otp_codes WHERE expires_at < now() - interval '1 day';` to keep the table small. If you switch to hashed codes, widen the `code` column (e.g. `varchar(64)` for sha256 hex).

## 6. Integrations

### Resend email (swappable) — `src/lib/email.ts`

All email delivery is behind one function, so swapping providers (Postmark, SES, SendGrid) only touches this file.

```ts
import { Resend } from "resend";

const resend = new Resend(process.env.RESEND_API_KEY);

export async function sendOtpEmail(email: string, code: string) {
  await resend.emails.send({
    from: "<YourApp> <noreply@yourdomain.com>",
    to: email,
    subject: `${code} is your <YourApp> verification code`,
    html: `
      <div style="font-family: sans-serif; max-width: 400px; margin: 0 auto; padding: 40px 20px;">
        <h2 style="color: #111; margin-bottom: 8px;">Your verification code</h2>
        <p style="color: #666; margin-bottom: 24px;">Enter this code to sign in to <YourApp>:</p>
        <div style="background: #f5f5f5; border-radius: 8px; padding: 20px; text-align: center; margin-bottom: 24px;">
          <span style="font-size: 32px; font-weight: bold; letter-spacing: 8px; color: #111;">${code}</span>
        </div>
        <p style="color: #999; font-size: 13px;">This code expires in 10 minutes. If you didn't request this, ignore this email.</p>
      </div>
    `,
  });
}
```

**To swap providers:** keep the `sendOtpEmail(email, code)` signature and replace the body. The `from` address domain must be verified with whatever provider you use.

### NextAuth session creation — `src/lib/auth.ts`

A Credentials provider whose `authorize()` runs `verifyOtp`, then find-or-creates the user. Session strategy is JWT (default for Credentials); the user id is copied onto the token and exposed on `session.user.id`.

```ts
import NextAuth from "next-auth";
import CredentialsProvider from "next-auth/providers/credentials";
import { db } from "./db";
import { users } from "./db/schema";
import { eq } from "drizzle-orm";
import { verifyOtp } from "./otp";

export const { handlers, signIn, signOut, auth } = NextAuth({
  trustHost: true,
  providers: [
    CredentialsProvider({
      name: "OTP",
      credentials: {
        email: { label: "Email", type: "email" },
        code: { label: "Code", type: "text" },
      },
      async authorize(credentials) {
        if (!credentials?.email || !credentials?.code) return null;
        const email = (credentials.email as string).toLowerCase().trim();
        const code = (credentials.code as string).trim();

        const valid = await verifyOtp(email, code);
        if (!valid) return null;

        // Find or create the user on first successful login
        let user = await db.query.users.findFirst({ where: eq(users.email, email) });
        if (!user) {
          const [newUser] = await db
            .insert(users)
            .values({ email, authProvider: "otp" })
            .returning();
          user = newUser;
        }

        return { id: user.id, email: user.email, name: user.name };
      },
    }),
  ],
  callbacks: {
    async session({ session, token }) {
      if (token?.sub) session.user.id = token.sub;
      return session;
    },
    async jwt({ token, user }) {
      if (user) token.sub = user.id;
      return token;
    },
  },
  pages: { signIn: "/login" },
});
```

> Wire the handlers in `src/app/api/auth/[...nextauth]/route.ts` with `export const { GET, POST } = handlers;` and wrap the app in NextAuth's `SessionProvider`.

## 7. Dependencies

### npm packages

| Package | Required? | Purpose |
|---------|-----------|---------|
| `next` (App Router) | Required | routing, route handlers, server components |
| `next-auth` (v5 / Auth.js) | Required | session creation via Credentials provider |
| `resend` | Optional* | email delivery (swap for any provider) |
| `drizzle-orm` | Required | DB queries (`eq`, `and`, `gt`, `gte`, `desc`) |
| `pg` / Postgres driver | Required | database connection used by `db` |
| `react` / `react-dom` | Required | login UI |

\* `resend` is only required if you keep Resend as the provider; the email layer is swappable.

UI components referenced (`Button`, `Input`, `Label`, `Card*`) are shadcn/ui primitives — replace with any component library.

### Environment variables

| Var | Required? | Purpose |
|-----|-----------|---------|
| `DATABASE_URL` | Required | Postgres connection string for `otp_codes` + `users` |
| `AUTH_SECRET` / `NEXTAUTH_SECRET` | Required | signs NextAuth JWT session tokens |
| `RESEND_API_KEY` | Required (if using Resend) | authenticates the email send; without it, OTP emails fail (500) |
| `NEXTAUTH_URL` | Optional | needed in some deploys; `trustHost: true` covers most |

### Hardcoded config to externalize

These live as constants in `src/lib/otp.ts` and the email module — consider moving to env/config:

| Constant | Default | Notes |
|----------|---------|-------|
| `BYPASS_EMAIL` | a single QA address | **Set to a value you control or disable in prod.** A bypass email accepts any 6-digit code without an inbox. |
| `OTP_EXPIRY_MINUTES` | 10 | code lifetime |
| `MAX_OTPS_PER_HOUR` | 3 | server rate limit per email |
| resend cooldown | 60s | client-side, in the login page |
| `from` address | `noreply@yourdomain.com` | must be a verified sender domain |

## 8. Porting Checklist

- [ ] Create the `otp_codes` table and `idx_otp_codes_email` index (Section 5).
- [ ] Ensure a `users` table exists with at least `id`, unique `email`, and `auth_provider`.
- [ ] Add the OTP service module (`generateOtp`, `isBypassEmail`, `checkRateLimit`, `createOtp`, `verifyOtp`).
- [ ] (Recommended) switch to a CSPRNG for code generation and **hash codes** before storing.
- [ ] Implement `sendOtpEmail(email, code)` against your email provider; verify the sender domain.
- [ ] Set `RESEND_API_KEY` (or your provider's key) and the `from` address.
- [ ] Add `POST /api/auth/send-otp` (normalize, bypass, rate-limit, create, send).
- [ ] Add `POST /api/auth/verify-otp` for non-NextAuth clients (optional if web-only).
- [ ] Configure NextAuth Credentials provider with the `authorize()` → `verifyOtp` → find-or-create-user flow.
- [ ] Set `AUTH_SECRET` / `NEXTAUTH_SECRET` and mount `[...nextauth]` route handlers + `SessionProvider`.
- [ ] Build the two-step login page (email → code), with numeric-only code input and submit gated on 6 digits.
- [ ] Add the 60s resend cooldown and "Change email" reset.
- [ ] Redirect already-authenticated users away from `/login`.
- [ ] **Change `BYPASS_EMAIL` to an address you own, or disable the bypass in production.**
- [ ] Add a cleanup job to purge expired/used `otp_codes` rows.
- [ ] Confirm raw codes are never written to logs.
- [ ] Test: happy path, expired code, reused code, wrong code, rate-limit (4th request in an hour → 429), and the bypass email.

---

*Built by [Quantana](https://quantana.in). MIT licensed.*
