# Newsletter Campaigns (SendGrid + First-Party Tracking)

## Feature Specification for Porting

This document describes the newsletter campaign builder: a multi-step wizard that designs an email in a WYSIWYG editor, targets an audience of CRM contacts, sends via [SendGrid](https://sendgrid.com), and measures engagement with **self-hosted, first-party open/click tracking** (no reliance on the ESP's tracking pixels). It also covers post-send analytics, CRM-integrated insights, tokenised unsubscribe, and image offload to S3.

It is a companion to the other feature specs in this library and assumes a multi-tenant (`workspace_id`-scoped) Postgres backend and a React frontend.

---

## 1. Overview

Users compose and send email newsletters to their CRM contacts through a four-step wizard:
- **Template** — pick a starting point (Blank, Announcement, Newsletter, Promotion)
- **Design** — build the email in an Unlayer WYSIWYG editor (drag/drop blocks, hosted image uploads, auto-save)
- **Configure** — set subject/preview/from-name, choose an audience (all / by tag / individuals / mixed), and optionally schedule
- **Review** — preview desktop/mobile, run a test send, then launch (or schedule)

Every send:
- Expands the audience into per-recipient rows, de-duplicated per contact
- Renders the compiled HTML, injects a **first-party open pixel** and **HMAC-signed click-redirect links** (when tracking is enabled), and explicitly **disables SendGrid's own tracking**
- Offloads any inline `data:` images to S3 before sending
- Records delivery, opens, clicks, bounces, and unsubscribes — from both the first-party endpoints and the SendGrid event webhook

After a send, users get an analytics view (open/click rates, an engagement timeline, and a per-recipient status table) plus CRM insight cards (warm leads, at-risk contacts, revenue attribution).

Safety rails: **test mode is ON by default** (sends only to workspace members), disabling it requires typing `BYPASS`, and any audience over 5 recipients requires typing `BLAST` to confirm.

---

## 2. User Flow

```
Sidebar → "Newsletters"  (/newsletters)
        │
        ▼
┌─────────────────────────────────────────────┐
│  Newsletter Dashboard (NewsletterList)       │
│  • Campaign cards (draft / sent / scheduled) │
│  • Sender identity + tracking toggle         │
│  • Audience metrics band                     │
│  • AI insight cards                          │
│  [ + New Newsletter ]                         │
└─────────────────────────────────────────────┘
        │
        ▼   4-step wizard (NewsletterWizard)
┌──────────┬──────────┬───────────┬──────────┐
│ Template │  Design  │ Configure │  Review  │
└──────────┴──────────┴───────────┴──────────┘
     │          │           │           │
     │          │           │           │
  pick a     Unlayer     audience +   preview +
 template    editor      subject +    test send +
 (creates   (auto-save   schedule      launch
  draft)     30s, S3
             image up)
                                          │
        ┌─────────────────────────────────┤
        │                                  │
        ▼ Test send (default ON)           ▼ Real send (test OFF → type "BYPASS")
   only workspace members            full audience (>5 → type "BLAST")
   subject prefixed "[TEST]"                │
   no tracking, no recipient rows           ▼
                                   POST /api/newsletters/{id}/send
                                            │
                                            ▼  background thread
                              ┌───────────────────────────────────┐
                              │  send_newsletter_background()      │
                              │  a. load newsletter + snippets     │
                              │  b. resolve audience → contacts    │
                              │  c. INSERT newsletter_recipients   │
                              │  d. offload data: images → S3      │
                              │  e. per recipient (batch 50):      │
                              │      render → inject tracking →    │
                              │      disable ESP tracking → send   │
                              │      → mark sent + message id      │
                              │  f. status = 'sent'                │
                              └───────────────────────────────────┘
                                            │
      ┌─────────────────────────────────────┼─────────────────────────────┐
      ▼                                     ▼                             ▼
 open pixel hit                     click redirect hit            SendGrid webhook
 GET /api/e/o/{token}.gif           GET /api/e/c/{token}?u=&s=    POST /api/webhooks/sendgrid
 → opened_at                        → clicked_at + redirect       → open/click/bounce/unsub
                                            │
                                            ▼
                          Analytics (NewsletterAnalytics): rates,
                          opens/clicks timeline, recipient table
```

---

## 3. Frontend Components

### 3.1 Dashboard (`NewsletterList.jsx`)

| Aspect | Detail |
|--------|--------|
| **Route** | `/newsletters` (shown in sidebar to all workspace members — no role gate) |
| **Campaign cards** | Grid of cards; each shows a scaled iframe preview of `html_compiled`, subject, sent date, recipient count, and open%/click% |
| **Status badges** | Draft, Sending, Sent, Scheduled, Failed |
| **Sender identity panel** | Editable `newsletter_from_name` + `newsletter_from_email`, with a SendGrid domain-verification warning; saved on blur |
| **Tracking toggle** | "Track opens & clicks" checkbox bound to the `newsletter_tracking` workspace setting (haptic feedback) |
| **Audience metrics band** | Total contacts with email, priority contacts (mindshare+), and contacts needing attention (from `/api/mindshare/stats`) |
| **AI insights** | Cards for revenue attribution, warm leads, at-risk contacts, plus greyed-out "deferred" placeholders (from `/api/newsletters/insights`) |
| **Delete** | Draft-only, with a confirmation modal |

### 3.2 Wizard Shell (`NewsletterWizard.jsx`)

A thin container that renders a step indicator (Template → Design → Configure → Review) with back-navigation, and mounts the active step component. State (the draft newsletter id + working copy) is threaded through the steps.

### 3.3 Step 1 — Template (`newsletter/StepTemplate.jsx`)

| Aspect | Detail |
|--------|--------|
| **Templates** | Blank, Announcement (hero + CTA), Newsletter (header/content/footer), Promotion (image + title + CTA) |
| **Action** | Selecting a template calls `useCreateNewsletter()` to create a `draft`, then advances to Design |

### 3.4 Step 2 — Design (`newsletter/StepDesign.jsx`)

| Aspect | Detail |
|--------|--------|
| **Editor** | `react-email-editor` (Unlayer WYSIWYG), height `calc(100vh - 260px)` (min 500px) |
| **Auto-save** | Every 30s via `useUpdateNewsletter()` — persists Unlayer `design_json` + exported `html_compiled` |
| **Image upload** | Registers a custom Unlayer upload callback that `POST`s to `/api/newsletters/upload-image` with a Bearer token; returns a **hosted URL** (never a base64 data URI) |
| **Accepted images** | JPG / PNG / GIF / WebP, max 5 MB |
| **Advance** | "Next" runs `editor.exportHtml()` and saves `design_json` + `html_compiled` |

### 3.5 Step 3 — Configure (`newsletter/StepConfigure.jsx`)

| Aspect | Detail |
|--------|--------|
| **Fields** | Subject (required), preview text, from name |
| **Audience modes** | `all`, `tags`, `individuals`, `mixed` (four-button selector) |
| **Tag select** | Multi-select pills (shown for `tags`/`mixed`), from `GET /api/tags` |
| **Contact/email add** | Autocomplete contact search (`GET /api/contacts`, ≤8 results) + free-text email entry (Enter to add), shown as removable pills |
| **Count preview** | `useAudiencePreview(config)` shows the recipient count + sample names for non-individual modes |
| **Schedule** | "Send immediately" or "Schedule for later" via native `<input type="datetime-local">`, stored as ISO 8601 `scheduled_at` |
| **Persistence** | Last-used individual contacts cached in `localStorage` (`newsletter-last-individual-contacts`) |

### 3.6 Step 4 — Review (`newsletter/StepReview.jsx`)

| Aspect | Detail |
|--------|--------|
| **Preview** | Collapsible-header email preview with desktop (600px) / mobile (320px) width toggle |
| **Preflight checks** | Subject set ✅/❌, body designed ✅/❌, ≥1 recipient ✅/❌ |
| **Test mode toggle** | **ON by default** (orange indicator). ON → sends only to workspace members. Turning OFF requires typing `BYPASS` in a modal |
| **Test send** | "Send Test to Me" (or "Send Test to N emails") via `useTestNewsletter({ to })`; custom test emails cached in `localStorage` (`newsletter_test_emails`) |
| **BLAST guard** | For any real send/schedule with >5 recipients, a modal requires typing `BLAST` to proceed |
| **Actions** | Back · Schedule (if `scheduled_at` set) · Send Now / "Send to Members" (when test mode) |

### 3.7 Analytics (`NewsletterAnalytics.jsx`)

| Aspect | Detail |
|--------|--------|
| **Layout** | Left: iframe preview of `html_compiled`. Right: stat cards (Sent, Opened %, Clicked %, Bounced) |
| **Timeline** | Recharts `LineChart` of opens + clicks per day (from `/api/newsletters/{id}/analytics`) |
| **Recipient table** | Name/email, status badge (sent/opened/clicked/bounced), and engagement timestamp (from `/api/newsletters/{id}/recipients`) |

### 3.8 Unsubscribe Page (`UnsubscribePage.jsx`)

Public, no-auth confirmation page reached from the unsubscribe link in every email. Calls `GET /api/newsletters/unsubscribe?token=…&email=…` and renders loading / success / error states.

### 3.9 Component File Structure

```
components/
  NewsletterList.jsx              # Dashboard: cards, sender identity, tracking toggle, insights
  NewsletterWizard.jsx            # 4-step wizard shell + step indicator
  NewsletterAnalytics.jsx         # Post-send analytics (rates, timeline, recipient table)
  UnsubscribePage.jsx             # Public tokenised unsubscribe confirmation
  NewsletterPreview.jsx           # Legacy modal preview (pre-Unlayer)
  newsletter/
    StepTemplate.jsx              # Template picker → creates draft
    StepDesign.jsx                # Unlayer editor + hosted image upload + auto-save
    StepConfigure.jsx             # Subject/audience/schedule
    StepReview.jsx                # Preview + test send + launch guards
hooks/
  useNewsletters.js               # All React Query hooks (see §5)
```

---

## 4. Frontend Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `react-email-editor` | 1.8.0 | Unlayer WYSIWYG email editor (drag/drop blocks, `design_json` + `exportHtml()`) |
| `recharts` | 3.8.1 | Opens/clicks timeline `LineChart` on the analytics view |
| `@tanstack/react-query` | 5.91.2 | Data fetching + cache for all newsletter hooks (`useQuery`/`useMutation`) |
| `axios` | ^1.18.0 | HTTP client |
| `dompurify` | ^3.4.0 | HTML sanitisation for safe preview rendering |
| `lucide-react` | 0.453.0 | Icons (Send, etc.) |
| `react-hot-toast` | 2.6.0 | Toast notifications |
| `ios-haptics` | 0.1.4 | Haptic feedback on the tracking toggle / actions (mobile) |

> Email previews are rendered inside sandboxed `<iframe srcDoc>` at scale (0.42 for card thumbnails, full width for the review/analytics preview) — no server-side rendering of the HTML is needed.

---

## 5. Backend API

All routes below live in `routers/newsletters.py` unless noted. All are **auth-required (Bearer/session)** and **workspace-scoped**, except the three public endpoints called out explicitly (`unsubscribe`, `/api/e/o`, `/api/e/c`).

### 5.1 CRUD

| Method | Path | Notes |
|--------|------|-------|
| `GET` | `/api/newsletters?status=` | List newsletters (+ per-row `sent_count`, `opened_pct`, `clicked_pct`, `bounced_pct`) |
| `POST` | `/api/newsletters` | Create draft — body `{subject, hero_title, hero_body, hero_image_url?, snippets[]}` |
| `GET` | `/api/newsletters/{id}` | Full newsletter + `stats {total, delivered, opened, clicked, bounced, unsubscribed}` |
| `PUT` | `/api/newsletters/{id}` | Patch any of `subject, hero_*, preview_text, from_name, scheduled_at, design_json, html_compiled, audience_config, snippets`. Blocked when `status` ∈ {sent, sending} |
| `DELETE` | `/api/newsletters/{id}` | Draft-only |
| `POST` | `/api/newsletters/{id}/clone` | Duplicate into a new draft |

### 5.2 `POST /api/newsletters/{id}/send`

**Body:** `{ "test_mode": boolean }` (optional)

Validates `status == 'draft'`, flips it to `sending`, and launches `send_newsletter_background(...)` in a background thread. `test_mode: true` restricts the audience to workspace members.

**Response:** `{ "status": "sending", "message": "..." }`

### 5.3 Scheduling

| Method | Path | Body | Notes |
|--------|------|------|-------|
| `POST` | `/api/newsletters/{id}/schedule` | `{ scheduled_at }` | draft/scheduled only → `{status:"scheduled", scheduled_at}` |
| `POST` | `/api/newsletters/{id}/cancel` | — | scheduled only → reverts to `{status:"draft"}` |

### 5.4 `POST /api/newsletters/{id}/test`

**Body:** `{ "to": ["a@x.com", ...] }` (optional; defaults to the caller's email, max 10)

Sends preview emails with a `[TEST]` subject prefix. **No tracking, no `newsletter_recipients` rows.**

**Response:** `{ "status": "ok", "sent_to": [...] }`

### 5.5 Audience & analytics

| Method | Path | Notes |
|--------|------|-------|
| `GET` | `/api/newsletters/audience-preview?config=<json>` | `{count, sample[]}` for a `{mode, tag_ids[], contact_ids[], emails[]}` config |
| `GET` | `/api/newsletters/{id}/analytics` | `{sent, opened, clicked, bounced, unsubscribed, open_rate, ctr, opens_timeline[], clicks_timeline[]}` |
| `GET` | `/api/newsletters/{id}/recipients` | Per-recipient rows: `{id, contact_id, email, name, status, sent_at, opened_at, clicked_at, bounced_at, resend_message_id}` |
| `GET` | `/api/newsletters/insights` | Dashboard cards: `warm_leads`, `at_risk`, `attribution`, `deferred[]` (see `newsletter_insights.py`) |

### 5.6 `POST /api/newsletters/upload-image`

`multipart/form-data` with `file` (jpeg/png/gif/webp, ≤5 MB). Compresses to ≤1200px wide (JPEG q82 or PNG, whichever is smaller), uploads to S3, and returns `{ url, size, original_size }`.

### 5.7 `GET /api/newsletters/unsubscribe` (public)

Query `token` (HMAC) + `email`. **No auth, rate-limited 10/min.** Verifies the token and sets `contacts.unsubscribed_at = NOW()`.

### 5.8 First-party tracking endpoints (public)

| Method | Path | Behaviour |
|--------|------|-----------|
| `GET` | `/api/e/o/{token}.gif` | Returns a 1×1 transparent GIF (`no-store`). On a valid recipient token, sets `opened_at` if NULL. **Always 200**, even on error/invalid token |
| `GET` | `/api/e/c/{token}?u=<url>&s=<sig>` | Verifies `s == sign_url(u)` (anti open-redirect; http(s) only), 302-redirects to `u`, and on a valid token sets `clicked_at` (+ `opened_at` if NULL). Invalid signature → `400 {"detail":"invalid or unsigned link"}` |

### 5.9 `POST /api/webhooks/sendgrid` (public, signed)

Consumes SendGrid Event Webhook batches. ECDSA-verified via `EventWebhook.verify_signature` using `SENDGRID_WEBHOOK_PUBLIC_KEY`. Matches events to `newsletter_recipients` by `resend_message_id` and maps:

```python
NEWSLETTER_EVENT_MAP = {
    "open":              ("opened_at",       False),
    "click":             ("clicked_at",      False),
    "bounce":            ("bounced_at",      False),
    "dropped":           ("bounced_at",      False),
    "unsubscribe":       ("unsubscribed_at", True),   # also_update_contact
    "group_unsubscribe": ("unsubscribed_at", True),
}
```

When the third tuple element is `True`, `contacts.unsubscribed_at` is also set.

> The webhook is a **fallback / complement** to first-party tracking. With `newsletter_tracking` on, opens/clicks arrive from `/api/e/*`; the webhook still supplies bounces and unsubscribes.

---

## 6. Database Schema

```sql
CREATE TABLE newsletters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL,                         -- tenant scope
    subject         TEXT,
    hero_title      TEXT,                                  -- legacy (pre-Unlayer) content
    hero_body       TEXT,                                  -- legacy
    hero_image_url  TEXT,                                  -- legacy
    snippets        JSONB DEFAULT '[]',                    -- [{emoji, title, body}, ...] (legacy)
    status          TEXT DEFAULT 'draft',                  -- draft|sending|sent|scheduled|failed
    scheduled_at    TIMESTAMPTZ,
    sent_at         TIMESTAMPTZ,
    created_by      UUID,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    design_json     JSONB,                                 -- Unlayer editor state
    html_compiled   TEXT,                                  -- final rendered email HTML
    audience_config JSONB DEFAULT '{"mode":"all","tag_ids":[],"contact_ids":[]}'::jsonb,
    preview_text    TEXT,
    from_name       TEXT                                   -- overrides env default sender name
);
CREATE INDEX idx_newsletters_workspace ON newsletters(workspace_id);

CREATE TABLE newsletter_recipients (
    id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),  -- encoded in tracking tokens
    newsletter_id      UUID REFERENCES newsletters(id) ON DELETE CASCADE,
    contact_id         UUID,                               -- NULL for direct-email recipients
    workspace_id       UUID NOT NULL,
    email              TEXT,
    resend_message_id  TEXT,                               -- SendGrid X-Message-Id (webhook match)
    personalized_intro TEXT,
    status             TEXT DEFAULT 'pending',             -- pending|sent|delivered|opened|clicked|bounced
    opened_at          TIMESTAMPTZ,
    clicked_at         TIMESTAMPTZ,
    bounced_at         TIMESTAMPTZ,
    unsubscribed_at    TIMESTAMPTZ,
    sent_at            TIMESTAMPTZ,
    clicked_url        TEXT,
    created_at         TIMESTAMPTZ DEFAULT NOW(),
    updated_at         TIMESTAMPTZ
);
CREATE INDEX idx_nr_newsletter_status ON newsletter_recipients(newsletter_id, status);
CREATE INDEX idx_nr_resend_msg        ON newsletter_recipients(resend_message_id);
CREATE INDEX idx_nr_contact           ON newsletter_recipients(contact_id);
-- de-dupe a contact within a newsletter
CREATE UNIQUE INDEX uq_nr_newsletter_contact ON newsletter_recipients(newsletter_id, contact_id);

-- per-workspace settings; tracking toggle + sender identity live here as key/value rows
CREATE TABLE workspace_settings (
    workspace_id UUID NOT NULL,
    key          TEXT NOT NULL,        -- 'newsletter_tracking' | 'newsletter_from_email' | 'newsletter_from_name'
    value        JSONB NOT NULL DEFAULT '{}'
);
```

> Unsubscribe state lives on the `contacts` table (`contacts.unsubscribed_at`), and is honoured by the audience query at send time (`unsubscribed_at IS NULL AND (status IS NULL OR status != 'do_not_contact')`).

---

## 7. Integrations

### 7.1 SendGrid (delivery)

- Sends via `SendGridAPIClient(SENDGRID_API_KEY).send(Mail(...))`.
- **ESP tracking is explicitly disabled** per message so the first-party pixel/links are the source of truth:
  ```python
  ts = TrackingSettings()
  ts.click_tracking = ClickTracking(enable=False, enable_text=False)
  ts.open_tracking  = OpenTracking(enable=False)
  message.tracking_settings = ts
  ```
- The message id is read from the `X-Message-Id` response header and stored in `newsletter_recipients.resend_message_id` for webhook correlation.
- Sender identity is resolved per send via `resolve_from_email(workspace_id)` → `workspace_settings` keys `newsletter_from_email` / `newsletter_from_name`, falling back to env defaults.

### 7.2 S3 / Vultr Object Storage (images)

- Both editor uploads (`/upload-image`) and inline `data:` URIs found in `html_compiled`/`design_json` at send time are offloaded.
- **Key pattern:** `newsletter/{uuid}.{ext}` · **ACL:** `public-read`.
- `sanitize_data_uris_in_html()` runs **once per send** (result persisted back to `newsletters.html_compiled`). On upload failure at send time it **strips** the broken data URI (blank image beats a multi-MB payload); autosave is failure-tolerant and keeps the token.
- Compression: LANCZOS resize to `MAX_EMAIL_WIDTH=1200`, transparency → PNG, else min(JPEG q82, PNG); animated GIFs pass through untouched.

### 7.3 First-party open/click tracking (`services/email_tracking.py`)

Self-hosted tracking with tamper-proof, PII-free tokens. Enabled per workspace via `workspace_settings.newsletter_tracking` (default **off**).

**Token = recipient id, HMAC-signed:**
```python
_SECRET = os.environ["JWT_SECRET"]; _SEP = "~"

def make_recipient_token(recipient_id: str) -> str:      # {payload}~{sig}
    payload = _b64e(str(recipient_id).encode())          # urlsafe b64 of the UUID
    return f"{payload}{_SEP}{_sig(payload.encode())}"    # sig = HMAC-SHA256[:18], urlsafe b64
```

**URL construction:**
```python
def open_pixel_url(base_url, token):                     # <img> src
    return f"{base_url.rstrip('/')}/api/e/o/{token}.gif"

def click_url(base_url, token, target):                  # signed redirect
    q = urllib.parse.urlencode({"u": target, "s": sign_url(target)})
    return f"{base_url.rstrip('/')}/api/e/c/{token}?{q}"
```

**Injection at send time** (`inject_tracking`): rewrites every `href="http(s)://…"` through `click_url` (skipping `/api/e/` and any `skip_substrings`, e.g. the unsubscribe link) and appends the open pixel before `</body>`.

#### Environment Variables

| Variable | Purpose | Default |
|----------|---------|---------|
| `SENDGRID_API_KEY` | SendGrid API key for delivery | — (required to send) |
| `SENDGRID_WEBHOOK_PUBLIC_KEY` | ECDSA key to verify the event webhook | — (required for webhook) |
| `NEWSLETTER_FROM_EMAIL` | Default sender email | `noreply@mevak.in` |
| `NEWSLETTER_FROM_NAME` | Default sender name | `Mevak AI` |
| `FRONTEND_URL` | Base URL for unsubscribe + first-party tracking links | `https://mevak.in` |
| `JWT_SECRET` | HMAC secret for tracking + link-signing tokens | — (required for tracking) |
| `UNSUBSCRIBE_SECRET` | HMAC secret for unsubscribe tokens | random if unset |
| `S3_BUCKET` / `AWS_S3_ENDPOINT` | Image offload bucket + endpoint (Vultr/AWS) | — (S3 optional) |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `AWS_REGION` | S3 credentials | region `us-east-1` |
| `ALPHA_MODE` | If set, suppresses real sends (staging safety) | unset |

#### Tracking source: first-party vs SendGrid webhook

| Signal | `newsletter_tracking = true` | `newsletter_tracking = false` |
|--------|------------------------------|-------------------------------|
| Opens | First-party `/api/e/o/{token}.gif` | SendGrid `open` webhook event |
| Clicks | First-party signed `/api/e/c/{token}` redirect | SendGrid `click` webhook event |
| Bounces | SendGrid `bounce`/`dropped` webhook | SendGrid `bounce`/`dropped` webhook |
| Unsubscribes | Tokenised link + SendGrid `unsubscribe` webhook | Tokenised link + SendGrid `unsubscribe` webhook |
| Open-rate caveat | Approximate — Apple Mail pre-fetches images | Approximate — same reason |
| Toggling mid-flight | Only affects **not-yet-sent** emails | — |

---

## 8. Data Collected Per Send

| Data Point | Source | Auto/Manual |
|------------|--------|-------------|
| Subject / preview / from-name | User input (Configure) | Manual |
| Email design + compiled HTML | Unlayer editor | Manual |
| Audience config (mode, tags, contacts, emails) | User selection | Manual |
| Recipient list (contact_id/email) | Audience resolution at send | Auto |
| `resend_message_id` | SendGrid `X-Message-Id` | Auto |
| Sent timestamp | Server `NOW()` on send | Auto |
| Opened at | First-party pixel or SendGrid webhook | Auto |
| Clicked at (+ clicked url) | First-party redirect or SendGrid webhook | Auto |
| Bounced at | SendGrid webhook | Auto |
| Unsubscribed at | Tokenised link or SendGrid webhook | Auto |

---

## 9. Porting Checklist

- [ ] **Postgres tables** — create `newsletters`, `newsletter_recipients` (with the partial unique index on `(newsletter_id, contact_id)`), and a `workspace_settings` key/value table
- [ ] **`contacts.unsubscribed_at`** — column exists and is honoured by the audience query
- [ ] **SendGrid account** — verify a sending domain; set `SENDGRID_API_KEY`, `NEWSLETTER_FROM_EMAIL`, `NEWSLETTER_FROM_NAME`
- [ ] **Disable ESP tracking on every message** — `ClickTracking(enable=False)` + `OpenTracking(enable=False)` so first-party wins
- [ ] **Capture `X-Message-Id`** into `newsletter_recipients.resend_message_id`
- [ ] **SendGrid Event Webhook** — enable signed events, set `SENDGRID_WEBHOOK_PUBLIC_KEY`, implement `POST /api/webhooks/sendgrid` with ECDSA verification + `NEWSLETTER_EVENT_MAP`
- [ ] **First-party tracking service** — `make_recipient_token`/`verify_recipient_token` (HMAC over the recipient UUID) and `sign_url`/`verify_url`; set `JWT_SECRET`
- [ ] **Open pixel endpoint** — `GET /api/e/o/{token}.gif` returns a 1×1 GIF, `no-store`, always 200, sets `opened_at` if NULL
- [ ] **Click redirect endpoint** — `GET /api/e/c/{token}?u=&s=` verifies the URL signature (http(s) only), 302-redirects, sets `clicked_at`
- [ ] **`inject_tracking`** — rewrite `href`s through the signed redirect (skip unsubscribe + `/api/e/`) and append the pixel before `</body>`
- [ ] **Per-workspace tracking toggle** — `workspace_settings.newsletter_tracking`, read at send time; default off
- [ ] **Tokenised unsubscribe** — `GET /api/newsletters/unsubscribe?token=&email=` (public, rate-limited), sets `contacts.unsubscribed_at`
- [ ] **Send pipeline** — background thread: load → resolve audience → INSERT recipients (`ON CONFLICT DO NOTHING`) → offload data-URI images → batch(50) render+inject+send → mark sent → status `sent`
- [ ] **Audience resolution** — modes `all|tags|individuals|mixed`; base filter excludes unsubscribed + `do_not_contact`; de-dupe per contact
- [ ] **Test send path** — `[TEST]` prefix, no tracking, no recipient rows, ≤10 addresses
- [ ] **S3 image offload** — `/api/newsletters/upload-image` (≤5 MB, compress to ≤1200px) + `sanitize_data_uris_in_html` at send (`newsletter/{uuid}.{ext}`, public-read); strip-on-failure at send, keep-on-failure on autosave
- [ ] **Frontend wizard** — `react-email-editor` (Unlayer) with a custom hosted-image upload callback; auto-save `design_json` + `html_compiled` every 30s
- [ ] **Configure step** — subject/preview/from-name, 4-mode audience selector, tag + contact + direct-email pickers, `audience-preview` count
- [ ] **Review step guards** — test mode ON by default, `BYPASS` to disable, `BLAST` confirmation for >5 recipients
- [ ] **Analytics view** — rates + Recharts opens/clicks timeline + per-recipient status table
- [ ] **Sender identity + tracking toggle UI** — on the dashboard, bound to `workspace_settings`, with a domain-verification warning

---

<!-- Companion to the ShipFactory feature spec library — https://github.com/vishalquantana/shipfactory -->

## About Us

We are [Quantana](https://quantana.com.au), an AI-first design and development agency working with Fortune 500s to build bespoke AI solutions and provide the audit and training needed to ensure success. [Click here to learn more](https://quantana.com.au).

## License

MIT
