# Automated Blog & Content Engine (GEO-Optimized, AI-Generated)

A drop-in, fully-automated content marketing system. It **discovers topics**
from real search data, **generates GEO-optimized articles** (plus a matching
data-visualization illustration) with an LLM, **stores them as MDX files** with
typed metadata, **renders them** with rich structured data (Article / FAQ /
Breadcrumb / Speakable JSON-LD), then **commits, indexes, and notifies** — with
zero human in the loop. A database queue lets a human override the AI with
hand-picked topics when desired.

> **GEO = Generative Engine Optimization** — content written to be cited by AI
> Overviews / ChatGPT / Perplexity, not just ranked by classic SEO. Every design
> choice below (direct-answer first paragraph, FAQ schema, Speakable selectors,
> citable benchmarks) serves that goal.

---

## 1. Overview

One cron-triggered script produces one publish-ready article end to end:

1. **Topic** — pop a human-queued topic from the DB, else discover one from live
   SERP data (People-Also-Ask questions + trending keyword suggestions).
2. **Content** — an LLM returns a single JSON object: the MDX body, a typed meta
   block (title, slug, excerpt, TL;DR, FAQs, takeaways, related slugs), and a
   self-contained SVG illustration component.
3. **Files** — write the `.mdx`, append the meta entry, write the illustration
   component, and auto-register it in the two registries the renderer reads.
4. **Ship** — git commit + push, submit the URL to the search Indexing API, ping
   sitemaps, and send a chat notification.

Content types are pluggable. The reference implementation ships four:
`insights` (short opinion), `guides` (long how-to), `compare` (comparison), and
`learn` (glossary). Each has its own length, tone, and prompt instruction.

---

## 2. Pipeline Flow

```
                      ┌─────────────────────────────────────────┐
   cron (daily) ─────▶│  generate-content script (orchestrator)  │
                      └─────────────────────────────────────────┘
                                       │
                 ┌─────────────────────┴─────────────────────┐
                 ▼                                            ▼
        ┌──────────────────┐                      ┌────────────────────────┐
        │ DB topic queue?  │── yes ──▶ use it ──▶ │  pick content type     │
        │ (manual override)│                      │  (by weekday schedule) │
        └──────────────────┘                      └────────────────────────┘
                 │ no                                          │
                 ▼                                             │
        ┌──────────────────────────────┐                      │
        │ Topic discovery (SERP API)   │                      │
        │  75% People-Also-Ask gaps    │                      │
        │  25% trending keyword sugg.  │                      │
        └──────────────────────────────┘                      │
                 │                                             │
                 └────────────────┬────────────────────────────┘
                                  ▼
                  ┌────────────────────────────────┐
                  │  LLM generation (single JSON)  │
                  │   mdxContent + meta + svg      │
                  └────────────────────────────────┘
                                  │
                                  ▼
        ┌───────────────────────────────────────────────────┐
        │  Write & register 5 things:                       │
        │   1. content/<type>/<slug>.mdx                    │
        │   2. content/<type>/meta.ts        (append entry) │
        │   3. components/illustrations/<Name>.tsx          │
        │   4. lib/illustrations.ts          (register map) │
        │   5. app/<type>/[slug]/page.tsx    (register MDX) │
        └───────────────────────────────────────────────────┘
                                  │
            ┌─────────────────────┼─────────────────────┐
            ▼                     ▼                     ▼
   git commit + push     Indexing API + sitemap    chat notification
                          ping (Google / Bing)        (Telegram)
```

**Wall-clock:** one article ≈ a single LLM call + a handful of file writes +
one git push. Runs unattended on a schedule.

---

## 3. Content Generation

### 3.1 Topic discovery (SERP data)

Two mechanisms, chosen by weighted random (≈75% / 25%):

| Source | API call | Returns |
|--------|----------|---------|
| **People-Also-Ask gaps** | SERP "organic / live / advanced" for a seed keyword | Natural-language questions real users ask |
| **Trending keywords** | "keyword suggestions / live" for the vertical | Rising keywords in your niche |

- Auth: provider Basic auth (`LOGIN` + `PASSWORD`).
- Localize results (set a location code, e.g. country/city) so benchmarks match
  your market.
- Maintain a **seed keyword list** for your domain; discovery expands from it.
- The reference uses DataForSEO; any SERP/keyword API (Semrush, Ahrefs,
  Serpapi) works — you only need "questions people ask" + "trending terms".

> **Don't have a SERP API?** Fall back to a static topic list, an editorial
> backlog file, or have the LLM brainstorm questions from your seed keywords.
> The DB queue (§6) already supports a 100%-manual workflow.

### 3.2 Content type & scheduling

A `ContentType` enum drives length, tone, and category. The reference schedule:

| Day | Type | Length | Shape |
|-----|------|--------|-------|
| Mon–Thu | `insights` | 600–900 w | Direct answer → numbered points; provocative, benchmark-led |
| Fri (rotating) | `guides` | 1000–1500 w | Direct answer + benchmark → bold numbered steps |
| Fri (rotating) | `compare` | 800–1200 w | Recommendation first → comparison table → "Which should you choose?" |
| Fri (rotating) | `learn` | 500–800 w | First sentence = definition → one worked example |

Each type maps to one or more display `category` labels and a per-type
instruction string injected into the prompt.

### 3.3 The generation prompt (the crown jewel)

A single LLM call returns **pure JSON** (no code fences) with three parts:
`mdxContent`, `meta`, `illustration`. The prompt enforces GEO rules:

```text
WRITING STYLE (GEO-optimized for AI Overview citations):
- CRITICAL: The FIRST paragraph must directly answer the topic in 2-3 sentences
  with specific data. No hooks, no stories, no preambles. AI models extract the
  first substantive paragraph — make it citation-worthy.
- Be factual and specific — real numbers, currency amounts, %, timeframes.
- Every H2 heading must match a natural search query
  ("How to reduce keyword waste on X", not "The keyword problem").
- Include ≥3 specific, citable data points per post.
- Numbered action lists with BOLD step names (AI Overviews extract these).
- Include 2-3 internal links in markdown format: [text](/learn/slug).
- Subtly position <YourProduct> as the solution where relevant — not salesy.
- Do NOT include any frontmatter / YAML header in the MDX content.

EXISTING SLUGS (do NOT reuse): <comma-separated list>

Return a JSON object with this exact structure:
{
  "mdxContent": "...",                  // pure MDX, no frontmatter
  "meta": { ...see §4.2... },
  "illustration": { "componentName": "PascalCase", "svgCode": "full TSX" }
}
```

Robustness details that matter in production:
- **Strip code fences** before `JSON.parse` (LLMs wrap output in ```` ```json ````).
- **Validate** required fields (`mdxContent`, `meta.slug`, `meta.title`); throw
  on missing.
- **Dedupe slugs** — if the returned slug already exists, suffix it with a
  timestamp.
- Pass the **full list of existing slugs** into the prompt so the model avoids
  topic collisions.

Reference model: `gemini-3-flash-preview` via `@google/generative-ai`. Any
JSON-capable LLM (GPT, Claude, Gemini) works — request JSON output and keep the
fence-stripping guard.

### 3.4 The illustration sub-spec

The same call returns a self-contained SVG React component so every post ships
with a visual. The prompt pins an exact style so output is on-brand:

```text
For the illustration SVG component:
- export default a React FC accepting { className?: string }
- viewBox="0 0 400 200"
- Background: <rect fill="#18181b" rx="12">
- Cards/sections: fill="#27272a"
- Text: fill="#a1a1aa" (light) / "#71717a" (mid)
- Accents: Red #dc2626/#ef4444, Green #4ade80, Gold #d4a853
- fontFamily="Arial, sans-serif"
- Content = a DATA VISUALIZATION (bar chart, comparison, flow) of the post's
  key insight — clean dark-mode data viz, NOT clipart.
```

Swap the palette/viewBox to match your brand. The key idea: the LLM emits a
**registerable component**, not an image URL — no image hosting, infinitely
themeable, and crisp at any size.

---

## 4. Content Storage Format

### 4.1 Layout (file-based, no DB needed to render)

```
content/
  guides/    <slug>.mdx ...   meta.ts   (GUIDES_META registry)
  insights/  <slug>.mdx ...   meta.ts   (POSTS_META  registry)
  compare/   <slug>.mdx ...   meta.ts   (COMPARE_META registry)
  learn/     <slug>.mdx ...   meta.ts   (LEARN_META  registry)
components/illustrations/<Name>.tsx       (one SVG component per post)
lib/
  illustrations.ts   ILLUSTRATIONS: Record<slug, Component>
  content.ts         getAllContent / getContentBySlug / getContentByCategory
  types.ts           ContentMeta interface
app/<type>/[slug]/page.tsx                (renderer + MDX_COMPONENTS map)
```

### 4.2 Metadata schema

MDX files carry **no frontmatter** — all metadata lives in typed `meta.ts`
registries, which gives you compile-time safety and easy cross-referencing.

```ts
interface ContentMeta {
  title: string;
  slug: string;
  excerpt: string;          // 120–155 chars
  category: string;
  date: string;             // ISO YYYY-MM-DD
  author?: string;
  featured?: boolean;
  publishDate?: string;     // hide until this date (scheduling)
  readTime?: string;        // derived, e.g. "5 min read"

  // --- GEO fields ---
  tldr?: string;            // 2–3 sentence summary with data → renders a TL;DR box
  faqs?: { question: string; answer: string }[];  // → FAQPage JSON-LD
  takeaways?: string[];     // 4 actionable key points
  relatedSlugs?: string[];  // cross-links to other posts
}
```

A registry entry looks like:

```ts
export const GUIDES_META: Record<string, Omit<ContentMeta, "readTime">> = {
  "your-post-slug": {
    title: "How to … (50–65 chars, SEO-optimized)",
    slug: "your-post-slug",
    excerpt: "One-sentence description, 120–155 chars.",
    category: "Platform Deep-Dives",
    date: "2026-03-20",
    author: "Author Name",
    featured: true,
    tldr: "2–3 sentence summary with the single most citable benchmark.",
    relatedSlugs: ["other-slug-a", "other-slug-b"],
    faqs: [{ question: "…?", answer: "… with a specific number." }],
    takeaways: ["Action 1", "Number 2", "Point 3", "Point 4"],
  },
  // …more posts…
};
```

> **Why MDX-in-files, not a CMS table?** Static generation, version control as
> your audit log, no runtime DB dependency to render, and content ships in the
> same PR/commit as code. The DB is used only for the *topic queue* (§6).

---

## 5. Rendering & SEO

### 5.1 Route per content type

`app/<type>/[slug]/page.tsx` for each type. Each implements:

- `generateStaticParams()` — every slug → static page at build (SSG).
- `generateMetadata()` — per-post OpenGraph / Twitter tags from `meta`.
- **Static MDX imports + a `MDX_COMPONENTS` map** (`slug → Component`). Static
  imports are required for bundler/SSG compatibility; the auto-registration in
  §3 / file-writer keeps this map current.

Render skeleton (every type shares it):

```jsx
<article>
  <script type="application/ld+json">…Article schema…</script>
  <SpeakableSchema />                 {/* voice/AI-extractable selectors */}
  <BreadcrumbNav />                   {/* + BreadcrumbList JSON-LD */}
  <h1>{meta.title}</h1>
  <TldrBox>{meta.tldr}</TldrBox>      {/* .tldr-box → Speakable target */}
  <div className="prose"><MDXContent /></div>
  <FaqSection faqs={meta.faqs} />     {/* + FAQPage JSON-LD */}
  <KeyTakeaways items={meta.takeaways} />
  <AuthorBio />
  <ContentCTA />                      {/* product signup CTA */}
  <RelatedContent items={relatedItems} />
</article>
```

### 5.2 Structured data (the GEO payload)

| Schema | Where | Purpose |
|--------|-------|---------|
| `Article` | inline `<script type="application/ld+json">` | headline, dates, author/publisher Organization, `about`, `mentions`, `inLanguage` |
| `FAQPage` | `FaqSection` | each FAQ → `Question` + `acceptedAnswer`; rich snippets |
| `BreadcrumbList` | `BreadcrumbNav` | position/name/item per crumb |
| `SpeakableSpecification` | `SpeakableSchema` | CSS selectors (`h1`, `.tldr-box`, `.prose > p:first-of-type`) marked AI/voice-extractable |

### 5.3 Helper layer (`lib/content.ts`)

```ts
getAllContent(type): ContentMeta[]            // filters by publishDate, sorts date desc, derives readTime
getContentBySlug(type, slug): ContentMeta
getContentByCategory(type, category): ContentMeta[]
```

### 5.4 Sitemap & robots

- **`sitemap.ts`** — emit every slug with tiered `priority` (home 1.0 → hubs 0.8
  → posts 0.7 → glossary 0.6) and per-type `changeFrequency`.
- **`robots.ts`** — explicitly **allow AI crawlers**: GPTBot, ChatGPT-User,
  PerplexityBot, ClaudeBot, Google-Extended, Bingbot. (This is the GEO opt-in —
  don't block the bots you want citing you.)

---

## 6. Database (topic queue only)

The renderer needs no DB. One optional table lets a human pre-empt the AI with
hand-picked topics:

```sql
CREATE TABLE blog_topics (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  title           text NOT NULL,
  description     text NOT NULL,
  target_keywords text[] NOT NULL,
  category        text NOT NULL,   -- enum: strategy|data|guides|platform-deep-dives|updates
  source          text NOT NULL,   -- enum: ai|manual
  status          text NOT NULL,   -- enum: pending|approved|rejected|used
  created_at      timestamp DEFAULT now(),
  updated_at      timestamp DEFAULT now()
);
```

Orchestrator logic:

```sql
-- Step 0: prefer a human-approved manual topic
SELECT id, title, category FROM blog_topics
WHERE status = 'approved' AND source = 'manual'
ORDER BY created_at ASC LIMIT 1;
-- …generate from it…
UPDATE blog_topics SET status = 'used' WHERE id = $1;
```

If the queue is empty, fall through to SERP discovery (§3.1). An optional
`blog_posts` table can mirror published metadata if you later want a DB-backed
admin view, but it is not required to render.

---

## 7. Integrations

### 7.1 LLM (content generation)
- Reference: Google Gemini (`@google/generative-ai`, `gemini-3-flash-preview`).
- Contract: one prompt → one JSON object (`mdxContent` + `meta` + `illustration`).
- Guard: strip ```` ``` ```` fences, `JSON.parse`, validate required fields.

### 7.2 SERP / keyword API (topic discovery)
- Reference: DataForSEO (Basic auth). Endpoints: SERP organic advanced (PAA) +
  Labs keyword suggestions. Localize via a location code.

### 7.3 Search Indexing
- Submit each new URL to the **Indexing API** (Google) using a service-account
  JWT (`jose` to sign). **Ping sitemaps** to Google and Bing so the post is
  crawled within hours, not days.

### 7.4 Notifications
- Post a chat message (reference: Telegram bot) with title, TL;DR, and live URL
  on success.

### 7.5 Git
- `git add -A` → commit (`content: add "<title>"`) → `git push`. Content ships
  through the same VCS history as code; the commit log is your publishing audit
  trail.

---

## 8. Dependencies

| Package | Purpose |
|---------|---------|
| `@google/generative-ai` | LLM content generation (swap for your provider SDK) |
| `@next/mdx`, `@mdx-js/loader`, `@mdx-js/react` | MDX as first-class pages |
| `remark-gfm`, `remark-frontmatter`, `remark-mdx-frontmatter` | GFM tables + frontmatter handling |
| `next`, `react`, `react-dom` | framework + SSG (Next 16, React 19 in reference) |
| `reading-time` | derive `readTime` |
| `satori`, `sharp` | OG image generation / image optimization |
| `jose` | sign service-account JWT for the Indexing API |
| `pg` + `dotenv` | topic-queue DB access + env loading |
| `tailwindcss` + `@tailwindcss/typography` | `prose` article styling |

MDX wiring (`next.config.ts`):

```ts
const withMDX = createMDX({
  options: {
    remarkPlugins: [remarkGfm, remarkFrontmatter, [remarkMdxFrontmatter, { name: "meta" }]],
    rehypePlugins: [],
  },
});
```

Required environment variables:

```bash
LLM_API_KEY=…                     # content generation
SERP_API_LOGIN= / SERP_API_PASSWORD=   # topic discovery (optional)
GOOGLE_SERVICE_ACCOUNT_JSON=…     # Indexing API (or *_FILE path)
NOTIFY_BOT_TOKEN= / NOTIFY_CHAT_ID=    # notifications (optional)
DATABASE_URL=postgres://…         # topic queue (optional)
```

---

## 9. Data / Capability Matrix

| Capability | Source | Required? |
|------------|--------|-----------|
| Topic | DB queue (manual) → SERP API → static fallback | one of |
| Article body (MDX) | LLM | yes |
| Typed meta (title/slug/excerpt/tldr/faqs/takeaways/related) | LLM | yes |
| Illustration (SVG component) | LLM | yes |
| Article / FAQ / Breadcrumb / Speakable JSON-LD | renderer from meta | yes |
| Sitemap + robots (AI crawlers allowed) | static routes | yes |
| Indexing submission + sitemap ping | Indexing API | optional |
| Notification | chat bot | optional |
| Scheduling / drafts | `publishDate` field + DB `status` | optional |

---

## 10. Porting Checklist

**Storage & rendering**
- [ ] Create `content/<type>/` folders + a typed `meta.ts` registry per type.
- [ ] Define the `ContentMeta` interface (incl. GEO fields: tldr/faqs/takeaways/relatedSlugs).
- [ ] Wire MDX into the framework (remark-gfm + frontmatter plugins).
- [ ] Build the `[slug]` route: `generateStaticParams`, `generateMetadata`, `MDX_COMPONENTS` map.
- [ ] Add `lib/content.ts` helpers (publishDate filter, date sort, readTime).

**GEO layer**
- [ ] Build components: `TldrBox`, `FaqSection`, `BreadcrumbNav`, `SpeakableSchema`, `KeyTakeaways`, `RelatedContent`, `ContentCTA`, `AuthorBio`.
- [ ] Emit Article / FAQPage / BreadcrumbList / Speakable JSON-LD.
- [ ] `sitemap.ts` with tiered priorities; `robots.ts` allowing AI crawlers.

**Illustrations**
- [ ] `components/illustrations/` + `lib/illustrations.ts` registry (`slug → Component`).
- [ ] Pin an SVG style spec in your prompt (viewBox, palette, data-viz only).

**Generation pipeline**
- [ ] LLM client returning one JSON object; fence-strip + validate + slug-dedupe.
- [ ] Per-type instruction strings + category map + weekday schedule.
- [ ] File-writer: write MDX, append meta entry, write illustration, register in both registries.
- [ ] (Optional) SERP topic discovery; (optional) `blog_topics` DB queue with manual override.

**Ship**
- [ ] git commit + push step.
- [ ] (Optional) Indexing API submission + sitemap ping.
- [ ] (Optional) chat notification.
- [ ] Schedule via cron (e.g. daily) and confirm the existing-slugs list is passed to the prompt.

---

*Built by [Quantana](https://quantana.in). Adapt the domain (vertical, palette,
seed keywords, CTA) and the engine is product-agnostic. MIT licensed.*
