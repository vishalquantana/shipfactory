# In-App Feedback & Screenshot Bug Reporter

## Feature Specification for Porting

This document describes the right-click feedback/bug report feature with automatic screenshot capture, used for collecting user feedback and bug reports directly from the application.

---

## 1. Overview

Users can right-click anywhere in the app to open a context menu with three options:
- **Report a Bug** — auto-captures a screenshot of the current page
- **Request a Feature** — opens a blank feedback form
- **View submissions** — navigates to the feedback history page (`/feedback`)

Inside the modal, screenshots can be added three ways — a **full-page capture**, a **drag-to-select region capture**, or **upload/paste** — and every screenshot can be **marked up** in a built-in annotation editor (pen, rectangle, arrow, text) before submitting.

On mobile, a floating action button (FAB) provides the same entry point.

Submitted reports are stored in the database, uploaded to S3, emailed to the team, and filed as Jira tickets automatically.

---

## 2. User Flow

```
Right-click anywhere (or tap FAB on mobile)
        │
        ▼
┌──────────────────────────┐
│   Context Menu            │
│  ┌────────────────────┐  │
│  │ 🐛 Report a Bug    │  │
│  │ 💡 Request Feature │  │
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
| **Options** | "Report a Bug" (red bug icon), "Request a Feature" (amber lightbulb icon), "View submissions" (navigates to `/feedback`) |

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

`FeedbackMenu` is mounted at the app root level in `App.jsx`:
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

**Auth:** Required (Bearer token)

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
  "id": "uuid",
  "status": "submitted",
  "screenshot_urls": ["https://s3.../feedback/abc123.png"],
  "jira_key": "MAV-123"
}
```

**Backend processing pipeline:**
1. Upload each screenshot to S3 (`feedback/{uuid}.{ext}`, `public-read` ACL)
2. Insert row into `feedback` table
3. Send email notification to admin with embedded screenshot images
4. Create Jira ticket (Bug or Story) with labels and attached screenshots
5. Return response with feedback ID, S3 URLs, and Jira key

### 5.2 `POST /api/client-error` (Automatic Error Reporter)

Separate from user-initiated feedback — captures unhandled JS errors automatically.

**Auth:** Required (Bearer token)

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

**Backend:** Creates Jira Bug ticket with labels `["auto-bug", "mevak-ui"]`.

---

## 6. Database Schema

```sql
CREATE TABLE public.feedback (
    id              uuid DEFAULT gen_random_uuid() NOT NULL PRIMARY KEY,
    workspace_id    uuid NOT NULL,
    user_email      text,
    type            text NOT NULL,
    page_url        text,
    description     text NOT NULL,
    created_at      timestamptz DEFAULT now(),
    screenshot_url  text,            -- first screenshot (backwards compat)
    screenshot_urls text[],           -- all screenshots as array
    CONSTRAINT feedback_type_check CHECK (type IN ('bug', 'feature'))
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
- Subject: `[Mevak Bug Report] from user@example.com` or `[Mevak Feature Request] from ...`
- Body includes: type, reporter email, page URL, description, embedded screenshot images (max-width 600px)

### 7.3 Jira Integration
- **Bug reports** create issue type `Bug` with labels: `["user-feedback", "mevak-bug"]`
- **Feature requests** create issue type `Story` with labels: `["user-feedback", "mevak-feature"]`
- Summary: `[Mevak Bug Report] {description first 180 chars}`
- Screenshots are downloaded from S3 and attached to the Jira issue via the attachments API
- Automatic client errors create `Bug` issues with labels: `["auto-bug", "mevak-ui"]`

---

## 8. Data Collected Per Report

| Data Point | Source | Auto/Manual |
|------------|--------|-------------|
| Report type (bug/feature) | User selection | Manual |
| Description text | User input | Manual |
| Page URL | `window.location.href` | Auto |
| Screenshots (up to 5) | Auto-capture, region capture, upload/paste — optionally marked up | Both |
| User email | Auth session | Auto |
| Workspace ID | Auth session | Auto |
| Timestamp | Server `now()` | Auto |

### Automatic Error Reports (separate system)
| Data Point | Source |
|------------|--------|
| Error message | `window.onerror` / `unhandledrejection` |
| Stack trace | Error object |
| Page URL | `window.location.href` |
| User-Agent | Request header |
| Component stack | React error boundary (when available) |

---

## 9. Porting Checklist

To replicate this feature in another application:

- [ ] **Context menu interception** — Listen for `contextmenu` event on document, prevent default, show custom menu at cursor position
- [ ] **Screenshot capture library** — Use `html-to-image` (or alternative like `html2canvas`) to capture DOM as PNG
- [ ] **Modal overlay management** — Hide the feedback modal from the screenshot, then restore it
- [ ] **Region capture** — Full-viewport drag-rectangle overlay + crop the full-page capture to the selected rect (offset by scroll position)
- [ ] **Markup / annotation editor** — Full-screen `<canvas>` over the screenshot with pen/rectangle/arrow/text tools, color palette, undo/clear; flatten annotations back into the image on save
- [ ] **Cross-origin image filtering** — Skip external `<img>` elements that would taint the canvas
- [ ] **Capture-phase Escape handling** — Region overlay and markup editor must capture Escape so they don't close the modal underneath
- [ ] **HEIC/HEIF conversion** — Convert iOS photo formats to JPEG before upload
- [ ] **Clipboard paste support** — Listen for `paste` event, extract image items from `clipboardData`
- [ ] **File upload with size limit** — Accept `image/*,.heic,.heif`, enforce 10MB max
- [ ] **Image count limit** — Cap at 5 images per report with UI counter
- [ ] **Multipart form submission** — Convert data URLs to blobs, send as `multipart/form-data`
- [ ] **S3 upload on backend** — Store screenshots with public-read ACL, return URLs
- [ ] **Database table** — Create `feedback` table with type check constraint
- [ ] **Email notification** — Send HTML email with embedded screenshot images on submission
- [ ] **Issue tracker integration** — Create tickets (Jira/Linear/GitHub Issues) with attached screenshots
- [ ] **Automatic error capture** — Hook `window.onerror` + `unhandledrejection` for zero-effort bug detection
- [ ] **Mobile entry point** — FAB button or equivalent for touch devices without right-click
- [ ] **Haptic feedback** — Provide tactile feedback on mobile for captures, errors, and submission

---

<!-- Part of the ShipFactory feature spec library — https://github.com/vishalquantana/shipfactory -->

## About Us

We are [Quantana](https://quantana.com.au), an AI-first design and development agency working with Fortune 500s to build bespoke AI solutions and provide the audit and training needed to ensure success. [Click here to learn more](https://quantana.com.au).

## License

MIT
