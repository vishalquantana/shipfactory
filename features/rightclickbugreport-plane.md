# In-App Feedback & Screenshot Bug Reporter (Plane Integration)

## Feature Specification for Porting

This document describes the right-click feedback/bug report feature with automatic screenshot capture, integrated with [Plane](https://plane.so) for issue tracking. It is a drop-in replacement for the Jira variant (`rightclickbugreport.md`), with all issue-tracker references updated to Plane's REST API.

---

## 1. Overview

Users can right-click anywhere in the app to open a context menu with two options:
- **Report Bug** — auto-captures a screenshot of the current page
- **Suggest Feature** — opens a blank feedback form

On mobile, a floating action button (FAB) provides the same entry point.

Submitted reports are stored in the database, uploaded to S3, emailed to the team, and filed as Plane work items automatically.

---

## 2. User Flow

```
Right-click anywhere (or tap FAB on mobile)
        │
        ▼
┌──────────────────────┐
│   Context Menu        │
│  ┌─────────────────┐  │
│  │ 🐛 Report Bug   │  │
│  │ 💡 Suggest Feat. │  │
│  └─────────────────┘  │
└──────────────────────┘
        │
        ▼
┌──────────────────────────────────────┐
│         Feedback Modal                │
│                                       │
│  [Bug] / [Feature]  toggle      [X]  │
│                                       │
│  Page: /deals/pipeline                │
│                                       │
│  ┌────────────────────────────────┐  │
│  │  Auto-captured screenshot      │  │
│  │  (preview with remove button)  │  │
│  └────────────────────────────────┘  │
│                                       │
│  [📷 Capture Screen] [🖼 Upload Image]│
│  "2/5 images · paste with ⌘+V"       │
│                                       │
│  ┌────────────────────────────────┐  │
│  │  Describe the bug...           │  │
│  │                                │  │
│  └────────────────────────────────┘  │
│                                       │
│  [         Submit         ]           │
└──────────────────────────────────────┘
        │
        ▼
   "Thanks for your feedback!" ✓
   (auto-closes after 1.5s)
```

---

## 3. Frontend Components

### 3.1 Context Menu (`FeedbackMenu.jsx`)

| Aspect | Detail |
|--------|--------|
| **Trigger** | `contextmenu` event on `document` |
| **Positioning** | Fixed at cursor (x, y), clamped to viewport edges |
| **Dismiss** | Click outside, Escape key |
| **Options** | "Report Bug" (red bug icon), "Suggest Feature" (amber lightbulb icon) |

### 3.2 Feedback Modal

| Aspect | Detail |
|--------|--------|
| **Overlay** | Fixed fullscreen, `rgba(0,0,0,0.5)` + `backdrop-filter: blur(4px)` |
| **Max width** | 480px, centered |
| **Type toggle** | Pill-style toggle between Bug (red) and Feature (amber) |
| **Page URL** | Displays `window.location.pathname` |
| **Description** | Textarea, min-height 100px, resizable |
| **Submit** | Disabled until description is non-empty |
| **Success** | Checkmark + "Thanks for your feedback!" — auto-closes after 1500ms |
| **Dismiss** | X button, Escape key |

### 3.3 Screenshot System

| Capability | Implementation |
|------------|----------------|
| **Auto-capture on bug report** | Fires 200ms after modal opens via `html-to-image` (`toPng`) |
| **Manual capture** | "Capture Screen" button — hides modal overlay, captures, restores |
| **Upload from file** | File input accepting `image/*,.heic,.heif`, max 10MB |
| **Paste from clipboard** | Listens for `paste` event, extracts `image/*` items |
| **HEIC/HEIF conversion** | Auto-converts to JPEG via `heic2any` library (quality 0.85) |
| **Max images** | 5 per report |
| **Preview** | Horizontal scroll strip with thumbnails + remove buttons |
| **Filter** | Excludes modal overlay, context menu, and cross-origin `<img>` elements from capture |

**Screenshot capture options:**
```js
toPng(document.body, {
  cacheBust: true,
  pixelRatio: 1,
  skipFonts: true,
  filter: (node) => {
    if (node.id === 'feedback-modal-overlay') return false
    if (node.id === 'feedback-context-menu') return false
    if (node.tagName === 'IMG' && !node.src.startsWith(window.location.origin)) return false
    return true
  },
})
```

### 3.4 Mobile Entry Point (`MagicFAB.jsx`)

A floating action button includes a "Report a Bug" option with subtitle "Screenshot + description sent to team". It triggers the same `FeedbackMenu` component via an `externalOpen` prop.

### 3.5 Integration Point

`FeedbackMenu` is mounted at the app root level:
```jsx
<FeedbackMenu externalOpen={feedbackType} onExternalClose={() => setFeedbackType(null)} />
```

---

## 4. Frontend Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `html-to-image` | ^1.11.13 | DOM-to-PNG screenshot capture |
| `heic2any` | * | HEIC/HEIF to JPEG conversion for iOS photos |
| `lucide-react` | * | Icons |
| `ios-haptics` | * | Haptic feedback on mobile |

---

## 5. Backend API

### 5.1 `POST /api/feedback`

**Auth:** Required (session or bearer token)

**Content-Type:** `multipart/form-data`

**Form fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | `"bug"` or `"feature"` |
| `description` | string | Yes | User's description text |
| `page_url` | string | No | `window.location.href` from frontend |
| `screenshots` | File[] | No | Up to 5 image files, max 10MB each |

**Response:**
```json
{
  "id": "ulid",
  "status": "submitted",
  "screenshot_urls": ["https://s3.../feedback/abc123.png"],
  "plane_issue_id": "a0feb8fe-d899-4e0b-ab53-69d8c9aa90b2"
}
```

**Backend processing pipeline:**
1. Upload each screenshot to S3 (`feedback/{uuid}.{ext}`, `public-read` ACL)
2. Insert row into `feedback` table
3. Create Plane work item with `description_html` containing screenshots inline
4. Send email notification to admin with embedded screenshot images
5. Return response with feedback ID, S3 URLs, and Plane issue ID

### 5.2 `POST /api/client-error` (Automatic Error Reporter)

Captures unhandled JS errors automatically — separate from user-initiated feedback.

**Body (JSON):**
```json
{
  "message": "TypeError: Cannot read property 'foo' of undefined",
  "stack": "TypeError: ...\n  at Component (file.js:42)",
  "url": "https://app.example.com/deals",
  "timestamp": "2026-05-06T10:30:00.000Z"
}
```

**Frontend hooks:**
- `window.onerror` — catches synchronous errors
- `unhandledrejection` event — catches async/promise errors
- Truncates message to 1000 chars, stack to 2000 chars

---

## 6. Database Schema

```sql
CREATE TABLE feedback (
    id              TEXT PRIMARY KEY,           -- ULID
    user_id         TEXT REFERENCES users(id),
    user_email      TEXT,
    type            TEXT NOT NULL CHECK(type IN ('bug', 'feature')),
    description     TEXT NOT NULL,
    page_url        TEXT,
    screenshot_urls TEXT NOT NULL DEFAULT '[]', -- JSON array of S3 URLs
    plane_issue_id  TEXT,                       -- Plane work item UUID
    created_at      TEXT NOT NULL
);
```

---

## 7. Integrations

### 7.1 S3 Storage
- **Key pattern:** `feedback/{uuid}.{ext}`
- **ACL:** `public-read`
- **Max file size:** 10MB per image
- **Max count:** 5 images per report

### 7.2 Email Notification
- Sends HTML email to admin on each submission
- Subject: `[App Bug Report] from user@example.com` or `[App Feature Request] from ...`
- Body includes: type, reporter email, page URL, description, embedded screenshot images (max-width 600px)

### 7.3 Plane Integration

#### Environment Variables

| Variable | Description |
|----------|-------------|
| `PLANE_API_KEY` | Personal Access Token — generate at `{your-plane-url}/profile/api-tokens/` |
| `PLANE_BASE_URL` | e.g. `https://plane.quantana.top` |
| `PLANE_WORKSPACE_SLUG` | Found in the URL: `/{workspace-slug}/projects/` |
| `PLANE_PROJECT_ID` | UUID from project URL: `/projects/{uuid}/issues/` |

#### Getting State IDs

Before deploying, fetch your project's states to get the correct UUIDs:

```bash
curl -s "https://{plane-url}/api/v1/workspaces/{slug}/projects/{project-id}/states/" \
  -H "X-Api-Key: YOUR_PAT" | jq '.[] | {id, name, group}'
```

Map states to report types:
- **Bug reports** → use a `backlog` or `unstarted` group state
- **Feature requests** → use an `unstarted` group state

#### Creating a Work Item

```ts
// src/lib/plane.ts

const PLANE_BASE = process.env.PLANE_BASE_URL
const WORKSPACE  = process.env.PLANE_WORKSPACE_SLUG
const PROJECT_ID = process.env.PLANE_PROJECT_ID
const API_KEY    = process.env.PLANE_API_KEY

// State UUIDs — fetch once from /states/ endpoint and hardcode
const STATE_BUG_BACKLOG   = "YOUR_BACKLOG_STATE_UUID"
const STATE_FEATURE_TODO  = "YOUR_TODO_STATE_UUID"

export async function createPlaneIssue({
  type,
  description,
  pageUrl,
  screenshotUrls,
  reporterEmail,
}: {
  type: "bug" | "feature"
  description: string
  pageUrl?: string
  screenshotUrls: string[]
  reporterEmail?: string
}): Promise<string | null> {
  const isBug = type === "bug"
  const title = isBug
    ? `[Bug Report] ${description.slice(0, 180)}`
    : `[Feature Request] ${description.slice(0, 180)}`

  const screenshotHtml = screenshotUrls
    .map(url => `<img src="${url}" style="max-width:600px;border-radius:8px;margin:8px 0;" />`)
    .join("\n")

  const descriptionHtml = `
    <p><strong>Reporter:</strong> ${reporterEmail || "Anonymous"}</p>
    ${pageUrl ? `<p><strong>Page:</strong> <code>${pageUrl}</code></p>` : ""}
    <p><strong>Description:</strong></p>
    <p>${description.replace(/\n/g, "<br>")}</p>
    ${screenshotUrls.length ? `<p><strong>Screenshots:</strong></p>${screenshotHtml}` : ""}
  `.trim()

  const res = await fetch(
    `${PLANE_BASE}/api/v1/workspaces/${WORKSPACE}/projects/${PROJECT_ID}/work-items/`,
    {
      method: "POST",
      headers: {
        "X-Api-Key": API_KEY,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        name: title,
        description_html: descriptionHtml,
        state: isBug ? STATE_BUG_BACKLOG : STATE_FEATURE_TODO,
        priority: isBug ? "urgent" : "none",
      }),
    }
  )

  if (!res.ok) {
    console.error("Plane issue creation failed:", await res.text())
    return null
  }

  const data = await res.json()
  return data.id as string
}
```

#### Key Differences vs Jira

| Aspect | Jira | Plane |
|--------|------|-------|
| Auth header | `Authorization: Basic base64(email:token)` | `X-Api-Key: YOUR_PAT` |
| Issue type field | `issuetype: { name: "Bug" }` | Controlled via `state` UUID |
| Priority values | `highest`, `high`, `medium` | `urgent`, `high`, `medium`, `low`, `none` |
| Labels | `labels: ["user-feedback"]` | Not available in CE — use state/priority instead |
| Screenshot attachment | Separate `/attachments` API call | Embedded inline in `description_html` |
| Issue key format | `PROJECT-123` | UUID (access via `sequence_id` for display) |
| Base URL pattern | `https://your-org.atlassian.net/rest/api/3/issue` | `{plane-url}/api/v1/workspaces/{slug}/projects/{id}/work-items/` |

---

## 8. Data Collected Per Report

| Data Point | Source | Auto/Manual |
|------------|--------|-------------|
| Report type (bug/feature) | User selection | Manual |
| Description text | User input | Manual |
| Page URL | `window.location.href` | Auto |
| Screenshots (up to 5) | Auto-capture + manual upload/paste | Both |
| User email | Auth session | Auto |
| Timestamp | Server `now()` | Auto |
| Plane issue ID | API response | Auto |

---

## 9. Porting Checklist

- [ ] **Get Plane PAT** — `{plane-url}/profile/api-tokens/` → add as `PLANE_API_KEY` env var
- [ ] **Get workspace slug** — from URL after login: `/{slug}/projects/`
- [ ] **Get project UUID** — from project URL: `/projects/{uuid}/issues/`
- [ ] **Fetch state UUIDs** — `GET /api/v1/workspaces/{slug}/projects/{id}/states/` → hardcode backlog + todo UUIDs
- [ ] **Test issue creation** — POST a test work item, verify it appears in Plane, delete it
- [ ] **Context menu interception** — `contextmenu` event on document, prevent default, show custom menu
- [ ] **Screenshot capture** — `html-to-image` `toPng`, filter out modal/menu nodes and cross-origin images
- [ ] **Modal overlay management** — hide feedback modal during capture, restore after
- [ ] **HEIC/HEIF conversion** — `heic2any` to JPEG before upload
- [ ] **Clipboard paste support** — `paste` event → extract `image/*` from `clipboardData`
- [ ] **File upload with size limit** — accept `image/*,.heic,.heif`, enforce 10MB max
- [ ] **Image count limit** — cap at 5 with UI counter
- [ ] **Multipart form submission** — data URLs → blobs → `multipart/form-data`
- [ ] **S3 upload on backend** — `feedback/{uuid}.{ext}`, public-read ACL
- [ ] **Database table** — `feedback` table with `plane_issue_id TEXT` column
- [ ] **Plane work item** — POST to work-items API with `description_html` containing inline screenshot `<img>` tags
- [ ] **Email notification** — HTML email to admin with embedded screenshots on submission
- [ ] **Automatic error capture** — `window.onerror` + `unhandledrejection`
- [ ] **Mobile FAB** — floating button for touch devices without right-click

---

<!-- Part of the ShipFactory feature spec library — https://github.com/vishalquantana/shipfactory -->

## About Us

We are [Quantana](https://quantana.com.au), an AI-first design and development agency working with Fortune 500s to build bespoke AI solutions and provide the audit and training needed to ensure success. [Click here to learn more](https://quantana.com.au).

## License

MIT
