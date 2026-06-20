# Google Indexing API Integration

> After publishing content, push the new URL directly into Google's index instead of waiting days for the crawler to discover it. This feature signs a service-account JWT, calls the Google Indexing API (`urlNotifications:publish`), and pings the sitemap to Google and Bing — so fresh pages typically get crawled within hours, not days.

## 1. Overview

Search engines crawl new pages on their own schedule, which can mean a multi-day delay between publishing and indexing. The Google Indexing API lets you actively notify Google the moment a URL is created or updated, dramatically shortening time-to-index.

This integration ships as a small, dependency-light module (`google-indexing.ts`) with two public functions:

- `submitUrlToGoogle(url)` — authenticates via a Google Cloud service account and calls the Indexing API to publish a `URL_UPDATED` notification for a single URL.
- `pingSitemaps(sitemapUrl)` — fires sitemap-ping requests to Google and Bing as a complementary signal.

It is invoked as the final step of the publish pipeline (`<YourApp>`'s blog generator) and is intentionally **non-fatal**: if credentials are missing or the API errors, the publish still succeeds and the failure is logged.

## 2. Flow

```
  ┌─────────────────────┐
  │ Publish content     │   (write MDX/HTML, build sitemap, deploy)
  │  → postUrl          │
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐
  │ Load service account│   GOOGLE_SERVICE_ACCOUNT_FILE
  │  (file or inline)   │   or GOOGLE_SERVICE_ACCOUNT_JSON
  └──────────┬──────────┘
             │  (none → warn + skip, non-fatal)
             ▼
  ┌─────────────────────┐
  │ Sign JWT (RS256)    │   iss=client_email, scope=.../indexing,
  │  with `jose`        │   aud=oauth2.googleapis.com/token, exp=+1h
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐
  │ Exchange JWT for    │   POST oauth2.googleapis.com/token
  │  OAuth access_token │   grant_type=jwt-bearer
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐
  │ Submit URL          │   POST indexing.googleapis.com/v3/
  │  type=URL_UPDATED   │        urlNotifications:publish
  └──────────┬──────────┘
             │
             ▼
  ┌─────────────────────┐
  │ Ping sitemaps       │   GET google.com/ping?sitemap=...
  │  (Google + Bing)    │   GET bing.com/ping?sitemap=...
  └─────────────────────┘
```

## 3. Service-Account Auth

Google's Indexing API uses OAuth 2.0 with a service account. Rather than pulling in the full `googleapis` SDK, this module signs the assertion JWT directly with [`jose`](https://github.com/panva/jose) and exchanges it for a short-lived access token.

The JWT is signed with `RS256` using the service account's PKCS#8 private key. The `scope` must be the Indexing scope and the `aud` must be Google's token endpoint.

```ts
import { SignJWT, importPKCS8 } from "jose";

interface ServiceAccount {
  client_email: string;
  private_key: string;
}

async function getAccessToken(sa: ServiceAccount): Promise<string> {
  const now = Math.floor(Date.now() / 1000);
  const key = await importPKCS8(sa.private_key, "RS256");
  const jwt = await new SignJWT({
    iss: sa.client_email,
    scope: "https://www.googleapis.com/auth/indexing",
    aud: "https://oauth2.googleapis.com/token",
    iat: now,
    exp: now + 3600,
  })
    .setProtectedHeader({ alg: "RS256", typ: "JWT" })
    .sign(key);

  const res = await fetch("https://oauth2.googleapis.com/token", {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: `grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer&assertion=${jwt}`,
  });
  const data = await res.json();
  return data.access_token;
}
```

The service account credentials are loaded from either a file path or inline JSON, so the same code works in local dev (file) and CI/serverless (env var):

```ts
function loadServiceAccount(): ServiceAccount | null {
  // Try file path first, then inline JSON
  const filePath = process.env.GOOGLE_SERVICE_ACCOUNT_FILE;
  if (filePath) {
    const { readFileSync } = require("fs");
    return JSON.parse(readFileSync(filePath, "utf-8"));
  }
  const saJson = process.env.GOOGLE_SERVICE_ACCOUNT_JSON;
  if (saJson) {
    return JSON.parse(saJson);
  }
  return null;
}
```

## 4. URL Submission

With an access token in hand, submit the URL to the Indexing API. Use `URL_UPDATED` for new or changed pages (use `URL_DELETED` when a page is removed). If credentials are absent, the function warns and returns — never throws — so the publish pipeline stays resilient.

```ts
export async function submitUrlToGoogle(url: string): Promise<void> {
  const sa = loadServiceAccount();
  if (!sa) {
    console.warn("⚠ GOOGLE_SERVICE_ACCOUNT_FILE/JSON not set, skipping indexing");
    return;
  }
  const token = await getAccessToken(sa);

  const res = await fetch(
    "https://indexing.googleapis.com/v3/urlNotifications:publish",
    {
      method: "POST",
      headers: {
        Authorization: `Bearer ${token}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ url, type: "URL_UPDATED" }),
    }
  );

  if (!res.ok) {
    const err = await res.text();
    throw new Error(`Google Indexing API error: ${res.status} ${err}`);
  }
  console.log(`✓ Submitted to Google Indexing: ${url}`);
}
```

**Caller (publish pipeline).** Wrap the calls so an indexing failure can never break a publish:

```ts
// Final step of the publish pipeline
console.log("[Step 6] Submitting to Google Indexing...");
try {
  await submitUrlToGoogle(postUrl);
  await pingSitemaps(`${SITE_URL}/sitemap.xml`);
} catch (e) {
  console.error("[Indexing] Failed (non-fatal):", (e as Error).message);
}
```

> **Quota note:** The Indexing API allows ~200 URL submissions per project per day by default. For bulk re-indexing, throttle requests (e.g. a `200ms` delay between calls).

## 5. Sitemap Ping

Sitemap pinging is a lightweight, no-auth complement to the Indexing API — a single GET request per search engine. Both pings are best-effort: failures are logged but never thrown.

```ts
export async function pingSitemaps(sitemapUrl: string): Promise<void> {
  const endpoints = [
    `https://www.google.com/ping?sitemap=${encodeURIComponent(sitemapUrl)}`,
    `https://www.bing.com/ping?sitemap=${encodeURIComponent(sitemapUrl)}`,
  ];
  for (const endpoint of endpoints) {
    try {
      await fetch(endpoint);
      console.log(`✓ Pinged: ${endpoint}`);
    } catch (e) {
      console.error(`✗ Failed to ping: ${endpoint}`, e);
    }
  }
}
```

> The Indexing API officially only supports `JobPosting` and `BroadcastEvent` structured data, but in practice it reliably accelerates crawling of any page. Sitemap pings broaden coverage to Bing and act as a fallback signal for all page types.

## 6. Google Cloud Setup

One-time setup to provision credentials and authorize the account against your verified property in Search Console:

1. **Create a Google Cloud project** at <https://console.cloud.google.com> (or reuse an existing one).
2. **Enable the Indexing API** for the project: APIs & Services → Library → search "Web Search Indexing API" → **Enable**.
3. **Create a service account:** IAM & Admin → Service Accounts → **Create Service Account**. Give it a name (e.g. `<yourapp>-indexer`). No project-level IAM role is required for the Indexing API.
4. **Generate a key:** open the service account → Keys → **Add Key → Create new key → JSON**. Download the JSON file (it contains `client_email` and `private_key`). Store it securely — this is your credential.
5. **Verify your site** in [Google Search Console](https://search.google.com/search-console) (if not already verified).
6. **Grant the service account Owner access in Search Console:** Search Console → Settings → **Users and permissions → Add user** → paste the service account's `client_email` → set permission to **Owner**. (The Indexing API requires *Owner*, not Restricted/Full.)
7. **Provision the credential** to your app via env var or file (see §7).

## 7. Dependencies

**npm:**

```bash
npm install jose
```

`jose` is the only runtime dependency (used for `SignJWT` + `importPKCS8`). Everything else uses the built-in `fetch` and Node's `fs`.

**Environment variables:**

| Variable | Required | Description |
|---|---|---|
| `GOOGLE_SERVICE_ACCOUNT_FILE` | One of these two **required** | Absolute path to the downloaded service-account JSON key file. Preferred for local/server deploys. Checked first. |
| `GOOGLE_SERVICE_ACCOUNT_JSON` | One of these two **required** | The full service-account JSON as an inline string. Preferred for CI / serverless where files are awkward. Used if `GOOGLE_SERVICE_ACCOUNT_FILE` is unset. |

> If **neither** is set, `submitUrlToGoogle` logs a warning and skips silently — indexing is treated as optional, not a hard dependency. The site URL / sitemap URL (e.g. `https://<yourapp>.com/sitemap.xml`) is supplied by the caller, not via env.

## 8. Porting Checklist

- [ ] `npm install jose`
- [ ] Create Google Cloud project and enable the Web Search Indexing API
- [ ] Create a service account and download its JSON key
- [ ] Verify the site in Search Console and add the service account `client_email` as an **Owner**
- [ ] Copy `google-indexing.ts` into your project (`scripts/lib/` or equivalent)
- [ ] Set `GOOGLE_SERVICE_ACCOUNT_FILE` **or** `GOOGLE_SERVICE_ACCOUNT_JSON`
- [ ] Replace the hardcoded site/sitemap URL with your own domain
- [ ] Call `submitUrlToGoogle(postUrl)` + `pingSitemaps(sitemapUrl)` as the last step of your publish flow
- [ ] Wrap the calls in try/catch so indexing failures are non-fatal
- [ ] For bulk re-indexing, add a ~200ms delay between submissions to respect the daily quota
- [ ] Verify success in Search Console → URL Inspection (or the Indexing API `urlNotifications:getMetadata` endpoint)

---

*Built by [Quantana](https://quantana.in). MIT licensed.*
