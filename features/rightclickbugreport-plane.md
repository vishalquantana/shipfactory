# In-App Feedback & Screenshot Bug Reporter (Plane Integration)

## Feature Specification for Porting

This document describes the right-click feedback/bug report feature with automatic screenshot capture, integrated with [Plane](https://plane.so) for issue tracking. It is a drop-in replacement for the Jira variant (`rightclickbugreport.md`), with all issue-tracker references updated to Plane's REST API.

---

## 1. Overview

Users can right-click anywhere in the app to open a context menu with three options:
- **Report Bug** — auto-captures a screenshot of the current page
- **Suggest Feature** — opens a blank feedback form
- **View submissions** — navigates to the feedback history page (`/feedback`)

Inside the modal, screenshots can be added three ways — a **full-page capture**, a **drag-to-select region capture**, or **upload/paste** — and every screenshot can be **marked up** in a built-in annotation editor (pen, rectangle, arrow, text) before submitting.

On mobile, a floating action button (FAB) provides the same entry point.

Submitted reports are stored in the database, uploaded to S3, emailed to the team, and filed as Plane work items automatically.

---

## 2. User Flow

```
Right-click anywhere (or tap FAB on mobile)
        │
        ▼
┌──────────────────────────┐
│   Context Menu            │
│  ┌────────────────────┐  │
│  │ 🐛 Report Bug      │  │
│  │ 💡 Suggest Feature │  │
│  │ 📋 View submissions │  │
│  └────────────────────┘  │
└──────────────────────────┘
        │
        ▼
┌────────────────────────────────────────────┐
│            Feedback Modal                    │
│                                              │
│  [Bug] / [Feature]  toggle             [X]  │
│                                              │
│  Page: /deals/pipeline                       │
│                                              │
│  ┌────────────────────────────────────┐    │
│  │  Auto-captured screenshot          │    │
│  │  (preview — ✏️ markup · 🗑 remove) │    │
│  └────────────────────────────────────┘    │
│                                              │
│  [📷 Capture Screen][✂️ Capture Area][🖼 Upload]│
│  "2/5 images · paste with ⌘+V"              │
│                                              │
│  ┌────────────────────────────────────┐    │
│  │  Describe the bug...               │    │
│  └────────────────────────────────────┘    │
│                                              │
│  [             Submit             ]          │
└────────────────────────────────────────────┘
        │
        ├──► Capture Area → drag a rectangle → cropped screenshot
        │
        ├──► ✏️ Markup → full-screen editor (pen/rect/arrow/text)
        │                 → Save flattens annotations onto image
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
| **Options** | "Report Bug" (red bug icon), "Suggest Feature" (amber lightbulb icon), "View submissions" (navigates to `/feedback`) |

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

### 3.3 Screenshot System (`feedback/captureUtils.js`)

| Capability | Implementation |
|------------|----------------|
| **Auto-capture on bug report** | Fires 200ms after modal opens via `html-to-image` (`toPng`) |
| **Full-page capture** | "Capture Screen" button — hides modal overlay, captures, restores |
| **Region capture** | "Capture Area" button — drag-select a rectangle, then crop (see 3.4) |
| **Markup / annotation** | Pencil button on each preview opens a full-screen editor (see 3.5) |
| **Upload from file** | File input accepting `image/*,.heic,.heif` (multiple), max 10MB each |
| **Paste from clipboard** | Listens for `paste` event, extracts `image/*` items |
| **HEIC/HEIF conversion** | Auto-converts to JPEG via `heic2any` library (quality 0.85) |
| **Max images** | 5 per report |
| **Preview** | Horizontal scroll strip with thumbnails + per-image markup ✏️ and remove 🗑 buttons |
| **Filter** | Excludes modal overlay, context menu, region overlay, and cross-origin `<img>` elements from capture |

**Screenshot capture options** (`capturePageDataUrl`):
```js
toPng(document.body, {
  cacheBust: true,
  pixelRatio: 1,
  skipFonts: true,
  filter: (node) => {
    // Exclude the feedback widget's own UI and cross-origin images
    if (node.id === 'feedback-modal-overlay'
      || node.id === 'feedback-context-menu'
      || node.id === 'feedback-region-overlay') return false
    if (node.tagName === 'IMG' && node.src && !node.src.startsWith(window.location.origin)) return false
    return true
  },
})
```

### 3.4 Region Capture (`feedback/RegionSelectOverlay.jsx`)

A "Capture Area" button lets users grab just part of the screen instead of the full page.

| Aspect | Detail |
|--------|--------|
| **Trigger** | "Capture Area" button in the modal — hides the modal, mounts a full-viewport overlay |
| **Selection** | Pointer-drag a rectangle; everything outside the selection is dimmed with four `rgba(0,0,0,0.45)` panels (clip-path-free for robustness) |
| **Selection border** | 2px accent-colored outline around the live rectangle |
| **Hint** | "Drag to select an area · Esc to cancel" banner shown before dragging |
| **Cancel** | Escape (capture-phase, so the modal's own Esc handler never fires) or a drag smaller than 8×8px |
| **Crop** | On release, the full page is captured, then `cropDataUrl()` slices the selected rect out, offset by `scrollX/scrollY` |
| **Coordinate mapping** | Capture is `pixelRatio: 1`, so page-space coordinates map 1:1 onto image pixels |

```js
// Crop a full-page capture down to the selected viewport rect.
export async function cropDataUrl(dataUrl, rect, scrollX = window.scrollX, scrollY = window.scrollY) {
  const img = await loadImage(dataUrl)
  const sx = clamp(rect.x + scrollX, 0, img.naturalWidth - 1)
  const sy = clamp(rect.y + scrollY, 0, img.naturalHeight - 1)
  const sw = clamp(rect.w, 1, img.naturalWidth - sx)
  const sh = clamp(rect.h, 1, img.naturalHeight - sy)
  const canvas = document.createElement('canvas')
  canvas.width = sw; canvas.height = sh
  canvas.getContext('2d').drawImage(img, sx, sy, sw, sh, 0, 0, sw, sh)
  return canvas.toDataURL('image/png')
}
```

> **Known limitation:** `html-to-image` renders `position: fixed` elements at their *unscrolled* location, so selections over fixed elements on a scrolled page may crop slightly wrong pixels for those elements.

### 3.5 Markup / Annotation Editor (`feedback/ScreenshotAnnotator.jsx`)

A pencil button on each screenshot preview opens a full-screen editor for annotating it before submission. Annotations are drawn on an HTML `<canvas>` sized to the image's native pixels and flattened back into the screenshot on save — no extra libraries.

| Aspect | Detail |
|--------|--------|
| **Tools** | Pen (freehand), Rectangle, Arrow (with arrowhead), Text |
| **Colors** | 4-swatch palette — Red, Amber, Blue, Black |
| **Edit actions** | Undo (last shape), Clear (all shapes) |
| **Text tool** | Click to place a floating input; Enter or blur commits, Escape cancels just the text |
| **Line/font scaling** | `lineWidth = max(3, imgWidth/400)`, `fontSize = max(16, imgWidth/60)` — annotations stay proportional on hi-res captures |
| **Save** | Flattens canvas → PNG data URL; re-encodes as JPEG (quality 0.85) if the PNG would exceed 5MB |
| **Cancel** | Discards annotations; Escape (capture-phase) so the underlying modal stays open |
| **Coordinates** | Pointer events are converted to image-native coordinates so drawings line up regardless of the letterboxed display size |
| **Perf** | In-progress shape held in a ref (no re-render per `pointermove`); committed shapes mirrored in a ref so Save always reads the latest |

**Drawing primitives:**
```js
// Each shape is a plain object replayed on every redraw:
{ type: 'pen',   color, points: [{x, y}, ...] }
{ type: 'rect',  color, x, y, w, h }
{ type: 'arrow', color, x1, y1, x2, y2 }   // arrowhead computed from angle
{ type: 'text',  color, x, y, text }
```

### 3.6 Component File Structure

```
components/
  FeedbackMenu.jsx              # Context menu + modal + capture/upload/paste orchestration
  FeedbackPage.jsx              # "View submissions" history page (/feedback route)
  feedback/
    captureUtils.js             # capturePageDataUrl() + cropDataUrl()
    RegionSelectOverlay.jsx     # Drag-to-select region picker
    ScreenshotAnnotator.jsx     # Full-screen markup/annotation canvas editor
```

### 3.7 Mobile Entry Point (`MagicFAB.jsx`)

A floating action button includes a "Report a Bug" option with subtitle "Screenshot + description sent to team". It triggers the same `FeedbackMenu` component via an `externalOpen` prop.

### 3.8 Integration Point

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
| `lucide-react` | * | Icons (Bug, Lightbulb, X, Send, Loader2, Camera, Crop, Trash2, ImagePlus, Pencil, ClipboardList; editor: Pen, Square, MoveUpRight, Type, Undo2, Check) |
| `ios-haptics` | * | Haptic feedback on mobile (confirm, error) |

> The region picker and markup editor are built on the native `<canvas>` API — **no extra image-editing dependency** is required.

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

**Backend:** Creates a Plane work item for the error (priority `urgent`), with the message, stack, and page URL in `description_html`. (Plane CE has no labels — use a dedicated state/priority to distinguish auto-reported errors.)

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
| Screenshots (up to 5) | Auto-capture, region capture, upload/paste — optionally marked up | Both |
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
- [ ] **Screenshot capture** — `html-to-image` `toPng`, filter out modal/menu/region-overlay nodes and cross-origin images
- [ ] **Modal overlay management** — hide feedback modal during capture, restore after
- [ ] **Region capture** — full-viewport drag-rectangle overlay + crop the full-page capture to the selected rect (offset by scroll position)
- [ ] **Markup / annotation editor** — full-screen `<canvas>` over the screenshot with pen/rectangle/arrow/text tools, color palette, undo/clear; flatten annotations back into the image on save
- [ ] **Capture-phase Escape handling** — region overlay and markup editor must capture Escape so they don't close the modal underneath
- [ ] **View submissions page** — `/feedback` history route (`FeedbackPage.jsx`)
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
