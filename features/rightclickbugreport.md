# In-App Feedback & Screenshot Bug Reporter

## Feature Specification for Porting

This document describes the right-click feedback/bug report feature with automatic screenshot capture, used for collecting user feedback and bug reports directly from the application.

---

## 1. Overview

Users can right-click anywhere in the app to open a context menu with two options:
- **Report Bug** — auto-captures a screenshot of the current page
- **Suggest Feature** — opens a blank feedback form

On mobile, a floating action button (FAB) provides the same entry point.

Submitted reports are stored in the database, uploaded to S3, emailed to the team, and filed as Jira tickets automatically.

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
    // Exclude feedback UI and cross-origin images
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
| `lucide-react` | * | Icons (Bug, Lightbulb, X, Send, Loader2, Camera, Trash2, ImagePlus) |
| `ios-haptics` | * | Haptic feedback on mobile (confirm, error) |

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
| Screenshots (up to 5) | Auto-capture + manual upload/paste | Both |
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
- [ ] **Cross-origin image filtering** — Skip external `<img>` elements that would taint the canvas
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
