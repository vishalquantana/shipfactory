# Playwright Vendor-Portal Automation

> Some vendors don't ship an API. They give you a web portal, an email login, and an OTP. To pull data programmatically you have to *be a browser*: log in via OTP, keep the session alive, click "Generate Report", wait, then download the file. This feature is a portable template for driving any such portal headlessly with Playwright — persisting the logged-in session (`storageState`) so you log in once and reuse it for weeks of automated runs.

The example below integrates a quick-commerce ads portal (Instamart), but nothing here is portal-specific except the CSS selectors and URLs. The *pattern* — OTP login → persist `storageState` → scrape a table → download ready files, with a concurrent-session manager bridging the two-step OTP flow across two HTTP requests — is the reusable product.

---

## 1. Overview

A vendor exposes ad/analytics data only through a logged-in web portal. Reports are not downloadable on demand: you fill a form, click **Generate**, and the portal builds the file asynchronously. Minutes (or hours) later it appears in an "Available Reports" table with a **Download** link.

`<YourApp>` automates the whole loop:

1. **Connect** — user enters their portal email in `<YourApp>`. We open a headless browser, submit the email, and trigger the portal's OTP.
2. **Verify OTP** — user pastes the OTP into `<YourApp>`. We enter it in the *same live browser*, complete login, and capture the authenticated `storageState`.
3. **Persist** — `storageState` (cookies + localStorage) is uploaded to S3 and referenced from the DB. This is the durable artifact: future runs need no human.
4. **Generate** — a scheduled collector restores the session, fills the report form for each report type / date range, and clicks **Generate Report**, recording a `pending` row per request.
5. **Download** — on a later run, the collector scans the "Available Reports" table, matches ready files to `pending` rows, downloads them, uploads to S3, and marks them `ready`.

Two HTTP requests (connect, verify-otp) straddle a human typing an OTP. The live Playwright browser must survive between them — that is what the **concurrent session manager** (§5) solves.

---

## 2. Flow

```
  ┌─────────┐   POST /connect          ┌──────────────────────┐
  │  User   │ ───{ email }───────────► │  startLogin()        │
  │ browser │                          │  • launch chromium   │
  └─────────┘                          │  • fill email        │
       ▲                               │  • click "Send OTP"  │
       │  { sessionToken }             │  • wait for OTP input│
       │◄──────────────────────────────│  (browser STAYS open)│
       │                               └──────────┬───────────┘
       │                                          │ LoginSession
       │                                          ▼
       │                               ┌──────────────────────┐
       │                               │ pendingSessions Map   │
       │   (user reads OTP from email) │ token → {browser,...} │  ◄─ 5-min TTL
       │                               └──────────┬───────────┘
       │  POST /verify-otp                         │
       └──{ sessionToken, otp }──────────────────► │
                                       ┌───────────▼──────────┐
                                       │ completeLogin()      │
                                       │  • fill OTP          │
                                       │  • click "Verify"    │
                                       │  • waitForURL(/dash/)│
                                       │  • context.storageState()
                                       └───────────┬──────────┘
                                                   │ storageState JSON
                                                   ▼
                            ┌───────────────────────────────────────┐
                            │ S3: sessions/<vendor>/<id>/state.json  │
                            │ DB: platformConnections.sessionStateS3Key
                            └───────────────────────────────────────┘
                                                   │
        ░░░░░░░░░░░░░░░░░░░  scheduled collector run  ░░░░░░░░░░░░░░░░░░░
                                                   │ restore storageState
                                                   ▼
            ┌──────────────────┐          ┌──────────────────┐
            │ generateReports()│ ───────► │ downloadReady     │
            │ fill form +      │  later   │ Reports()         │
            │ "Generate"       │  run     │ scan table,       │
            │ → DB row=pending │          │ download → S3,    │
            └──────────────────┘          │ DB row=ready      │
                                          └──────────────────┘
```

---

## 3. OTP Login + Session Persistence

This is the key reusable pattern. Login is split into **two functions** because the OTP arrives out-of-band (email/SMS) and must be supplied by a human between two HTTP requests. `startLogin` returns the *live* Playwright session; `completeLogin` consumes it and returns the serialized `storageState`.

```typescript
// collector/<vendor>/login.ts
import { chromium, Browser, BrowserContext, Page } from "playwright";
import { PORTAL_LOGIN_URL } from "./config";

export interface LoginSession {
  browser: Browser;
  context: BrowserContext;
  page: Page;
}

/**
 * Step 1: Navigate to login, enter email, click "Send OTP".
 * Returns the LIVE session — the browser stays open awaiting OTP entry.
 */
export async function startLogin(email: string): Promise<LoginSession> {
  const browser = await chromium.launch({ headless: true });
  const context = await browser.newContext();
  const page = await context.newPage();

  await page.goto(PORTAL_LOGIN_URL, { waitUntil: "networkidle" });

  // ⚠ Selectors are VENDOR-SPECIFIC — customize per portal.
  await page.fill(
    'input[type="email"], input[placeholder*="email" i], input[name="email"]',
    email,
  );
  await page.click(
    'button:has-text("Send OTP"), button:has-text("Get OTP"), button:has-text("Continue")',
  );

  // Block until the OTP field renders (proof the portal accepted the email)
  await page.waitForSelector(
    'input[type="text"], input[placeholder*="OTP" i], input[maxlength="6"]',
    { timeout: 15000 },
  );

  return { browser, context, page };
}

/**
 * Step 2: Enter OTP, complete login, return storageState JSON for persistence.
 */
export async function completeLogin(session: LoginSession, otp: string): Promise<string> {
  const { context, page } = session;

  // Handle both single-input and per-digit OTP layouts.
  const otpInputs = await page.$$('input[maxlength="1"]');
  if (otpInputs.length >= 6) {
    for (let i = 0; i < 6; i++) await otpInputs[i].fill(otp[i]);
  } else {
    await page.fill(
      'input[type="text"], input[placeholder*="OTP" i], input[maxlength="6"]',
      otp,
    );
  }

  await page.click(
    'button:has-text("Verify"), button:has-text("Submit"), button:has-text("Login")',
  );

  // Landing on a post-login route is our success signal.
  await page.waitForURL(/\/(dashboard|reports|home)/, { timeout: 30000 });

  // Serialize cookies + localStorage. THIS is the durable credential.
  const storageState = await context.storageState();
  return JSON.stringify(storageState);
}

export async function closeSession(session: LoginSession): Promise<void> {
  try {
    await session.browser.close();
  } catch {}
}

/**
 * Cheap liveness check: load a protected page with the saved state.
 * If we're redirected to /login, the session has expired.
 */
export async function isSessionValid(storageStateJson: string): Promise<boolean> {
  const browser = await chromium.launch({ headless: true });
  try {
    const context = await browser.newContext({
      storageState: JSON.parse(storageStateJson),
    });
    const page = await context.newPage();
    await page.goto(PORTAL_LOGIN_URL.replace("/login", "/reports"), {
      waitUntil: "networkidle",
      timeout: 20000,
    });
    return !page.url().includes("/login");
  } finally {
    await browser.close();
  }
}
```

**Restoring the session in a collector run** — no human involved. Pull `storageState` from S3 into a temp file and hand it to `newContext`. After the run, write any refreshed state back (sliding-window sessions stay alive longer this way):

```typescript
// collector/<vendor>-runner.ts (excerpt)
const browser = await chromium.launch({ headless: true });

let storageStatePath: string | undefined;
if (authStateS3Key) {
  const sessionData = await getFromS3(authStateS3Key);
  storageStatePath = path.join(os.tmpdir(), `session-${connectionId}.json`);
  fs.writeFileSync(storageStatePath, sessionData);
}

const context = await browser.newContext(
  storageStatePath ? { storageState: storageStatePath } : {},
);
const page = await context.newPage();

const result = await collectReports(page, connectionId);

if (result.sessionExpired) {
  await db.update(platformConnections)
    .set({ status: "auth_expired", sessionHealth: "expired" })
    .where(eq(platformConnections.id, connectionId));
  // → user must re-run the OTP connect flow
} else {
  await db.update(platformConnections)
    .set({ lastCollectedAt: new Date(), status: "active", sessionHealth: "healthy" })
    .where(eq(platformConnections.id, connectionId));

  // Persist refreshed cookies so the session keeps sliding forward.
  if (authStateS3Key) {
    const updatedState = await context.storageState();
    await uploadToS3(authStateS3Key, JSON.stringify(updatedState), "application/json");
  }
}
```

---

## 4. Triggering & Downloading

Two phases against the same restored session. **Generate** fills the report form and records a `pending` row; **Download** (on a later run) scans the table and pulls files that are now ready.

### 4a. Trigger generation

For each report type, compute the date range needed (incremental since the last successful download), avoid re-triggering anything already `pending` within 24h, fill the form, and click **Generate Report**.

```typescript
// collector/<vendor>/generate-reports.ts
export async function generateReports(page: Page, connectionId: string): Promise<GenerateResult> {
  const result: GenerateResult = { triggered: 0, skipped: 0, errors: [] };
  const today = new Date();

  for (const reportConfig of REPORT_TYPES) {
    // Last successful download → start point for the next incremental pull.
    const lastDownload = await db.query.reportDownloads.findFirst({
      where: and(
        eq(reportDownloads.connectionId, connectionId),
        eq(reportDownloads.reportType, reportConfig.type),
        eq(reportDownloads.status, "ready"),
      ),
      orderBy: [desc(reportDownloads.endDate)],
    });

    // Don't double-trigger: skip if a fresh pending request already exists.
    const existingPending = await db.query.reportDownloads.findFirst({
      where: and(
        eq(reportDownloads.connectionId, connectionId),
        eq(reportDownloads.reportType, reportConfig.type),
        eq(reportDownloads.status, "pending"),
      ),
      orderBy: [desc(reportDownloads.createdAt)],
    });
    if (existingPending) {
      const ageMs = Date.now() - new Date(existingPending.createdAt).getTime();
      if (ageMs < 24 * 60 * 60 * 1000) { result.skipped++; continue; }
      // Stale (>24h, never appeared) → mark failed and re-trigger.
      await db.update(reportDownloads).set({ status: "failed" })
        .where(eq(reportDownloads.id, existingPending.id));
    }

    const lastEndDate = lastDownload ? new Date(lastDownload.endDate) : null;
    // Chunks a long backfill into <= maxDays windows (portal export limit).
    const ranges = calculateDateRanges(reportConfig, lastEndDate, today);

    for (const range of ranges) {
      await triggerReportGeneration(page, reportConfig, range.startDate, range.endDate);

      await db.insert(reportDownloads).values({
        connectionId,
        reportType: reportConfig.type,
        startDate: range.startDateRaw.toISOString().split("T")[0],
        endDate: range.endDateRaw.toISOString().split("T")[0],
        s3Key: s3ReportKey(connectionId, reportConfig.type, range.startDateRaw, range.endDateRaw),
        fileName: generateReportName(reportConfig.type),
        status: "pending",
      }).onConflictDoNothing(); // unique(connId, type, start, end)

      result.triggered++;
      await page.waitForTimeout(2000); // be polite between submissions
    }
  }
  return result;
}

// ⚠ Selectors are VENDOR-SPECIFIC.
async function triggerReportGeneration(
  page: Page, config: ReportTypeConfig, startDate: string, endDate: string,
): Promise<void> {
  const dropdown = await page.$('select, [class*="dropdown"], [role="listbox"]');
  if (dropdown) {
    await dropdown.click();
    await page.click(`text="${config.portalLabel}"`); // human-readable portal label
  }

  const datePickers = await page.$$('input[type="date"], [class*="date-picker"]');
  if (datePickers.length >= 2) {
    await datePickers[0].fill(startDate);
    await datePickers[1].fill(endDate);
  }

  await page.click('button:has-text("Generate Report")');
  await page.waitForTimeout(3000); // wait for "queued" confirmation
}
```

### 4b. Download ready files

Scan the "Available Reports" table, match each row to a `pending` DB record by filename, capture the download, upload to S3, and flip the row to `ready`.

```typescript
// collector/<vendor>/download-reports.ts
export async function downloadReadyReports(page: Page, connectionId: string): Promise<DownloadResult> {
  const result: DownloadResult = { downloaded: 0, errors: [] };

  const pendingReports = await db.query.reportDownloads.findMany({
    where: and(
      eq(reportDownloads.connectionId, connectionId),
      eq(reportDownloads.status, "pending"),
    ),
  });
  if (pendingReports.length === 0) return result;

  if (!page.url().includes("/reports")) {
    await page.click('a:has-text("Reports"), [href*="reports"]');
    await page.waitForTimeout(3000);
  }

  // ⚠ Table/row/cell selectors are VENDOR-SPECIFIC.
  const rows = await page.$$('table tbody tr, [class*="report"] [class*="row"]');

  for (const row of rows) {
    const reportName = await row
      .$eval('td:nth-child(1), [class*="name"]', (el) => el.textContent?.trim() || "")
      .catch(() => "");
    const downloadLink = await row.$(
      'a:has-text("Download"), button:has-text("Download"), [class*="download"]',
    );
    if (!downloadLink || !reportName) continue;

    for (const pending of pendingReports) {
      // Fuzzy filename match (portal name vs our generated name prefix).
      if (pending.fileName &&
          reportName.includes(pending.fileName.split("_").slice(0, 3).join("_"))) {
        try {
          // Playwright captures the browser download event.
          const downloadPromise = page.waitForEvent("download", { timeout: 30000 });
          await downloadLink.click();
          const download = await downloadPromise;

          const tmpPath = path.join(os.tmpdir(), `report-${pending.id}.csv`);
          await download.saveAs(tmpPath);

          const stats = fs.statSync(tmpPath);
          if (stats.size === 0) { result.errors.push(`Empty file for ${reportName}`); continue; }

          const s3Key = pending.s3Key ?? s3ReportKey(
            connectionId, pending.reportType,
            new Date(pending.startDate), new Date(pending.endDate),
          );
          await uploadToS3(s3Key, fs.readFileSync(tmpPath), "text/csv");

          await db.update(reportDownloads).set({
            status: "ready",
            s3Key,
            fileSizeBytes: stats.size,
            downloadedAt: new Date(),
          }).where(eq(reportDownloads.id, pending.id));

          result.downloaded++;
          try { fs.unlinkSync(tmpPath); } catch {}
        } catch (err) {
          result.errors.push(`Failed to download ${reportName}: ${err}`);
        }
      }
    }
  }
  return result;
}
```

---

## 5. Concurrent Session Management

The OTP flow spans **two separate HTTP requests** — `/connect` (sends OTP) and `/verify-otp` (submits OTP) — with a human in between. The live Playwright `Browser`/`Page` from step 1 must survive until step 2 supplies the code. A simple in-memory `Map`, keyed by an opaque token, bridges the gap, with a TTL so abandoned browsers get reaped instead of leaking processes.

```typescript
// src/lib/<vendor>-sessions.ts
import { LoginSession, closeSession } from "../../collector/<vendor>/login";

export interface PendingLoginEntry {
  session: LoginSession;   // live Playwright browser/context/page
  email: string;
  brandId: string;         // <YourApp> tenant scope — abstract as needed
  createdAt: number;
}

// token → live login session. Lives in the server process memory.
export const pendingSessions = new Map<string, PendingLoginEntry>();
export const SESSION_TIMEOUT_MS = 5 * 60 * 1000; // OTPs expire fast

/** Reap abandoned sessions; close their browsers to free RAM/processes. */
export function cleanupExpiredSessions() {
  const now = Date.now();
  for (const [token, entry] of pendingSessions) {
    if (now - entry.createdAt > SESSION_TIMEOUT_MS) {
      closeSession(entry.session);
      pendingSessions.delete(token);
    }
  }
}
```

**Notes & trade-offs**

- **Concurrency** — the `Map` is keyed by a random token, so many users can have in-flight OTP logins simultaneously, each owning its own headless browser. Cap parallelism if memory is tight (each Chromium is ~50–150 MB).
- **Single-process assumption** — this in-memory store only works on one server process. Behind multiple replicas, route `/connect` and `/verify-otp` to the same instance (sticky sessions) or externalize the live session (e.g. a dedicated single-replica "login worker"). The `storageState` itself is durable (S3); only the *5-minute live-browser window* is process-local.
- **Always close** — every exit path (success, timeout, OTP failure) must call `closeSession` or you leak a Chromium per failed login. Call `cleanupExpiredSessions()` at the top of `/connect`.

---

## 6. API Endpoints

Two POST endpoints, tenant-scoped (`requireBrandAccess` → adapt to your auth).

### `POST /api/.../<vendor>/connect`

Starts login, stashes the live session, returns an opaque `sessionToken`.

```typescript
const { email } = await req.json();
cleanupExpiredSessions();

const session = await startLogin(email);            // launches browser, sends OTP
const sessionToken = crypto.randomBytes(32).toString("hex");

pendingSessions.set(sessionToken, { session, email, brandId, createdAt: Date.now() });

return NextResponse.json({
  sessionToken,
  message: "OTP sent. Enter the code to complete connection.",
});
```

### `POST /api/.../<vendor>/verify-otp`

Looks up the live session, completes login, persists `storageState`, records the connection.

```typescript
const { sessionToken, otp } = await req.json();

const entry = pendingSessions.get(sessionToken);
if (!entry) return NextResponse.json({ error: "Session expired or invalid." }, { status: 410 });
if (entry.brandId !== brandId) return NextResponse.json({ error: "Brand mismatch" }, { status: 403 });
if (Date.now() - entry.createdAt > SESSION_TIMEOUT_MS) {
  await closeSession(entry.session);
  pendingSessions.delete(sessionToken);
  return NextResponse.json({ error: "Session timed out. Please restart." }, { status: 410 });
}

try {
  const storageStateJson = await completeLogin(entry.session, otp);
  const s3Key = `sessions/<vendor>/${brandId}/storage-state.json`;
  await uploadToS3(s3Key, storageStateJson, "application/json");
  const connection = await addConnection(brandId, entry.email, s3Key); // → platformConnections row
  return NextResponse.json({ success: true, connectionId: connection.id });
} finally {
  await closeSession(entry.session);
  pendingSessions.delete(sessionToken);
}
```

---

## 7. Database Schema

Two tables: one for the connection (holds the pointer to the persisted session), one to track each report request through its `pending → ready` lifecycle.

```typescript
// Connection: one row per (tenant, vendor). sessionStateS3Key is the linchpin.
export const platformConnections = pgTable("platform_connections", {
  id: uuid("id").primaryKey().defaultRandom(),
  brandId: uuid("brand_id").notNull().references(() => brands.id),
  platform: platformEnum("platform").notNull(),
  sessionStateS3Key: text("session_state_s3_key"),          // ← S3 path to storageState JSON
  lastCollectedAt: timestamp("last_collected_at"),
  lastSessionRefresh: timestamp("last_session_refresh"),
  status: connectionStatusEnum("status").default("learning"),     // active | auth_expired | ...
  sessionHealth: sessionHealthEnum("session_health").default("healthy"), // healthy | expired
  createdAt: timestamp("created_at").defaultNow().notNull(),
});

// One row per report request. Unique key prevents duplicate triggers.
export const reportDownloads = pgTable("report_downloads", {
  id: uuid("id").primaryKey().defaultRandom(),
  connectionId: uuid("connection_id").notNull().references(() => platformConnections.id),
  reportType: reportTypeEnum("report_type").notNull(),
  startDate: date("start_date").notNull(),
  endDate: date("end_date").notNull(),
  s3Key: text("s3_key"),
  fileName: text("file_name"),
  fileSizeBytes: integer("file_size_bytes"),
  status: downloadStatusEnum("status").default("pending").notNull(), // pending | ready | failed
  downloadedAt: timestamp("downloaded_at"),
  createdAt: timestamp("created_at").defaultNow().notNull(),
}, (table) => [
  uniqueIndex("uq_report_download").on(
    table.connectionId, table.reportType, table.startDate, table.endDate,
  ),
  index("idx_report_download_conn").on(table.connectionId, table.reportType),
]);
```

`status` lifecycle: `pending` (generation triggered) → `ready` (downloaded + in S3) → or `failed` (stale, never appeared in the table). The unique index on `(connectionId, reportType, startDate, endDate)` makes `onConflictDoNothing()` idempotent across overlapping runs.

---

## 8. Dependencies

| Dependency | Purpose |
|---|---|
| `playwright` (chromium) | Headless browser: login, form-fill, download capture |
| `@aws-sdk/client-s3` | Store `storageState` JSON + downloaded report files |
| `drizzle-orm` + Postgres | Connection + download tracking tables |
| Next.js route handlers | `connect` / `verify-otp` endpoints (any HTTP framework works) |
| `crypto` (Node built-in) | Opaque `sessionToken` generation |

Install the browser binary in CI/containers: `npx playwright install --with-deps chromium`.

**Environment variables**

```bash
# S3 (or any S3-compatible store)
S3_BUCKET=your-bucket
S3_REGION=...
S3_ACCESS_KEY_ID=...
S3_SECRET_ACCESS_KEY=...
# S3_ENDPOINT=...           # if using MinIO / R2 / Vultr Object Storage

# Database
DATABASE_URL=postgres://...

# Vendor portal (per integration)
PORTAL_LOGIN_URL=https://partner.<vendor>.com/login
PORTAL_REPORTS_URL=https://partner.<vendor>.com/reports
```

---

## 9. Porting Checklist

- [ ] **⚠ Customize ALL portal selectors per vendor.** Every `page.fill` / `page.click` / `page.$$` selector in §3–§4 is a guess for the example portal. Open the real portal in `headless: false`, inspect the actual email field, OTP inputs, generate button, report table rows, and download links. **This is where the bulk of porting effort goes.**
- [ ] Set `PORTAL_LOGIN_URL` / `PORTAL_REPORTS_URL` and the post-login URL regex in `completeLogin`'s `waitForURL`.
- [ ] Determine the OTP input layout (single field vs per-digit) and adjust `completeLogin`.
- [ ] Define your `REPORT_TYPES` (portal labels, max date-range per export, overlap days) and `calculateDateRanges` chunking.
- [ ] Adjust filename matching in `downloadReadyReports` to the portal's naming convention (or match on date columns / row order instead).
- [ ] Wire `addConnection` to write a `platformConnections` row with `sessionStateS3Key`.
- [ ] Pick a `SESSION_TIMEOUT_MS` matching the vendor's OTP validity window.
- [ ] Replace `requireBrandAccess` with your own auth/tenancy guard.
- [ ] Schedule the collector (cron / queue) and handle `auth_expired` by notifying the user to re-run the OTP connect flow.
- [ ] If running multiple server replicas, add sticky routing for `/connect`+`/verify-otp` or externalize the live login session (§5).
- [ ] Run the browser session through `isSessionValid` before each collection to fail fast on expiry.
- [ ] Set `headless: true` only after the selectors are verified in headed mode.
- [ ] Add a per-host concurrency cap so you don't hammer the vendor (and don't OOM on Chromium instances).

---

*Built by [Quantana](https://quantana.in). MIT licensed.*
