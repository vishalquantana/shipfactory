# Weekly Social Carousel Generator (Agent-Curated, HTML→PNG→Instagram)

A companion to the [Automated Blog & Content Engine](./automated-blog-content-engine.md).
Where that engine produces long-form articles, this produces a **weekly
multi-slide social carousel** — a "This Week In `<Vertical>`" roundup — and posts
it to Instagram. The slides are **styled HTML rendered to PNG by a headless
browser** (Playwright), so they're pixel-perfect, fully themeable, and need no
designer. The *content* (which news, what to advise) is **curated by an AI agent
in the loop**, under a strict "verified-and-sourced only" rule — never auto-
hallucinated.

> **Why this pattern beats canvas/image-libs:** you author each slide as a normal
> HTML/CSS template (web fonts, gradients, flexbox, your real logo) and screenshot
> it at 2× DPR. Anyone who can write CSS can restyle a slide; no SVG path math, no
> design tool, no image API.

---

## 1. Overview

The system has three separable parts:

1. **Curate (agent-in-the-loop)** — the AI agent gathers the week's real,
   sourced news for your vertical and produces a typed `WeeklyNewsData` object
   (a week label + N items, each with a headline, an actionable takeaway, and an
   emoji). **No automated LLM news-fetch** — the agent verifies each item
   against a real source before including it.
2. **Render (deterministic)** — `generateWeeklyNewsCarousel(data)` builds HTML
   for a **cover slide** (all headlines) + one **detail slide per item**
   ("What Happened" → "What You Should Do"), and screenshots each to a PNG buffer.
3. **Publish** — PNG slides are written to a dated folder, then
   `post-weekly-news <folder>` uploads them as one Instagram carousel via a
   social-posting API, and sends a chat notification.

```
  AI agent (curates + verifies)        Playwright (renders)         Social API
 ┌──────────────────────────┐      ┌───────────────────────┐    ┌──────────────┐
 │ WeeklyNewsData            │      │ cover slide (HTML)    │    │ upload N PNGs│
 │  weekLabel                │─────▶│ + detail slide × N    │───▶│ as 1 carousel│
 │  items[]: {headline,      │      │ → screenshot @2× DPR  │    │ + caption    │
 │    takeaway, emoji}       │      │ → PNG buffers         │    │ + hashtags   │
 └──────────────────────────┘      └───────────────────────┘    └──────────────┘
        (sourced only)              1080×1350, brand-styled        + chat notify
```

---

## 2. The Agent Brief (how the AI generates "this week's stuff")

This is the operational contract that lives in your repo's `CLAUDE.md` /
`AGENTS.md` so the agent produces a correct carousel every time:

- **Sourced-only, zero hallucination.** Every news item must be a real event the
  agent verified against a citable source this week. No projected, rumored, or
  invented headlines. (This is the single most important rule — social posts are
  public and a fabricated "news" item is a brand risk.)
- **Each item = `{ headline, takeaway, emoji }`.**
  - `headline` — what happened, factual, one line.
  - `takeaway` — **actionable advice for *your audience*** ("What you should do
    about it"), not a restatement of the news. This is the value-add that makes
    the post worth a follow.
  - `emoji` — one leading glyph for visual scanning.
- **Pick 4–6 items.** Enough to be a "roundup," few enough that each gets its own
  slide. Cover slide lists them all; detail slides expand each.
- **`weekLabel`** — e.g. `"WEEK OF APR 20, 2026"`, uppercased.
- **Output a single typed object**, then run the renderer. The agent does not
  touch styling — the template owns the look.

```ts
interface NewsItem { headline: string; takeaway: string; emoji: string }
interface WeeklyNewsData { weekLabel: string; items: NewsItem[] }
```

> Port note: this is deliberately **agent-in-the-loop, not a cron**. News
> curation needs judgment and source-checking; automating it invites
> hallucinated posts. Schedule a weekly *reminder* for the agent, not an
> unattended generate-and-post.

---

## 3. Rendering (HTML → PNG)

`scripts/lib/weekly-news-generator.ts` — pure, deterministic, no network except
web fonts.

**Canvas:** `1080 × 1350` (Instagram 4:5 portrait), `deviceScaleFactor: 2` so
exported PNGs are 2160×2700 — crisp on retina.

**Cover slide** (`coverSlideHtml`):
- Top accent gradient bar, radial background "glow" gradients.
- Uppercase `weekLabel`, big title ("This Week In `<Vertical>`" with an accent
  span), underline rule.
- A vertically-centered list of all headlines (emoji + text, dividers between).
- Footer: brand logo (inlined as a base64 data URI so the screenshot is self-
  contained), slide counter `1 / N+1`, "Swipe for takeaways →".

**Detail slides** (`detailSlideHtml`, one per item):
- A **rotating accent color** per slide (`ACCENT_COLORS` array, indexed by
  `i % colors.length`) — each gets its own main/glow/bg trio.
- "What Happened" tag → emoji → headline.
- Divider → "What You Should Do" label → **takeaway card** (tinted bg, accent
  left-border).
- Footer: `@handle` + week label, slide counter `i+2 / N+1`.

**Render loop:**

```ts
export async function generateWeeklyNewsCarousel(data: WeeklyNewsData): Promise<Buffer[]> {
  const browser = await chromium.launch();
  const context = await browser.newContext({
    viewport: { width: 1080, height: 1350 },
    deviceScaleFactor: 2,
  });
  const images: Buffer[] = [];
  images.push(await renderPage(context, coverSlideHtml(data)));            // slide 1
  for (let i = 0; i < data.items.length; i++)                              // slides 2..N+1
    images.push(await renderPage(context, detailSlideHtml(data.items[i], i, data.items.length, data.weekLabel)));
  await browser.close();
  return images;
}

async function renderPage(context, html): Promise<Buffer> {
  const page = await context.newPage();
  await page.setContent(html, { waitUntil: "networkidle" }); // waits for web fonts
  const png = await page.screenshot({ type: "png" });
  await page.close();
  return Buffer.from(png);
}
```

Production details that matter:
- **`waitUntil: "networkidle"`** so Google Fonts finish loading before the shot —
  otherwise text renders in a fallback font.
- **Inline the logo as a base64 data URI** (`readFileSync` → `data:image/png;base64,…`)
  so there are no external requests / broken images in the screenshot.
- **`esc()` all dynamic text** (`& < > "`) before interpolating into HTML — the
  agent's headlines/takeaways are untrusted input.
- One reused `BrowserContext`, one page per slide, closed each time — keeps memory flat.

---

## 4. Publishing

### 4.1 Folder convention

Rendered PNGs are written to a dated folder, sorted so order is stable:

```
scripts/weekly-news/week-2026-04-20/
  slide-1.png  slide-2.png  …  slide-6.png
```

### 4.2 Post script

`post-weekly-news <folder>` reads every `*.png` (sorted), then uploads them as a
single carousel:

```ts
const files = readdirSync(folder).filter(f => f.endsWith(".png")).sort();
const images = files.map(f => readFileSync(join(folder, f)));
const caption = `This Week In <Vertical> — your weekly roundup of what happened and what to do about it.\n\nSwipe through for the news and actionable takeaways.`;
const hashtags = ["…", "…"];   // niche + brand tags
await postCarouselToInstagram(images, caption, hashtags);
```

### 4.3 Social API call

Reference uses **Upload-Post** (multipart `photos[]` array → one IG carousel):

```ts
const API_BASE = "https://api.upload-post.com/api";
const formData = new FormData();
images.forEach((buf, i) =>
  formData.append("photos[]", new Blob([buf], { type: "image/png" }), `slide-${i+1}.png`));
formData.append("platform[]", "instagram");
formData.append("user", "<profile-handle>");
formData.append("description", `${caption}\n\n${hashtags.map(h => `#${h}`).join(" ")}`);

const res = await fetch(`${API_BASE}/upload_photos`, {
  method: "POST",
  headers: { Authorization: `Apikey ${process.env.UPLOAD_POST_API_KEY}` },
  body: formData,
});
// poll status: GET /api/uploadposts/status?request_id=<id>
```

- **No API key set → skip gracefully** (warn, return `{success:false}`) so local
  runs don't fail. The key gates only the publish step.
- On success/failure, send a **chat notification** (Telegram) with the request ID.
- Swap Upload-Post for Meta Graph API, Buffer, Ayrshare, etc. — the contract is
  just "array of images + caption → one carousel."

---

## 5. Dependencies

| Package | Purpose |
|---------|---------|
| `playwright` (chromium) | render HTML slides → PNG at 2× DPR |
| `dotenv` | load API keys / config |
| (fetch + FormData) | multipart upload to the social API (built into Node 18+) |
| Google Fonts (Inter) | slide typography via `@import` (loaded at render time) |

Environment:

```bash
UPLOAD_POST_API_KEY=…     # social publishing (optional — skip → render only)
NOTIFY_BOT_TOKEN= / NOTIFY_CHAT_ID=   # success/failure notification (optional)
```

**Required to run:** none for rendering (Playwright + a logo file). The API keys
gate only the publish + notify steps. So you can generate slides offline and
post manually if you prefer.

---

## 6. Porting Checklist

**Rendering**
- [ ] Add Playwright; pick canvas size for your platform (1080×1350 IG portrait, 1080×1080 square, 1080×1920 story).
- [ ] Write `coverSlideHtml` + `detailSlideHtml` templates in your brand styling; inline your logo as a base64 data URI.
- [ ] Define a rotating `ACCENT_COLORS` palette for detail slides.
- [ ] Implement `generateWeeklyNewsCarousel(data)` + a `renderPage` helper (`waitUntil:"networkidle"`, screenshot, close).
- [ ] `esc()` all agent-supplied text before interpolation.

**Agent brief**
- [ ] Put the §2 brief in your `CLAUDE.md`/`AGENTS.md`: sourced-only (no hallucinated news), `{headline, takeaway, emoji}` per item, 4–6 items, actionable takeaways.
- [ ] Define the `WeeklyNewsData` / `NewsItem` types.

**Publishing**
- [ ] Dated output folder convention (`week-YYYY-MM-DD/slide-N.png`).
- [ ] `post-weekly-news <folder>` script: read+sort PNGs → upload as one carousel.
- [ ] Wire a social-posting API (Upload-Post / Meta Graph / Buffer); skip gracefully if key unset.
- [ ] Caption + niche/brand hashtags; chat notification with request ID.
- [ ] Schedule a **weekly agent reminder** (not an unattended cron — curation needs a human-grade verify step).

---

*Built by [Quantana](https://quantana.in). Restyle the templates and swap the
vertical, palette, and social API — the render→post pipeline is product-agnostic.
MIT licensed.*
