# SEO/GEO Component Kit

> A drop-in kit of reusable React components and JSON-LD generators that make any content page **citable by AI**. GEO (Generative Engine Optimization) is the practice of structuring a page so that AI Overviews, ChatGPT, Perplexity, and Gemini can extract, quote, and cite it — not just rank it. This kit ships the presentation primitives (TL;DR box, FAQ, key takeaways, comparison table, breadcrumbs, related content, CTA) **paired with the structured-data schema** that those engines parse, so every content page becomes machine-readable and quotable by default.

This document covers **only** the reusable presentation + schema component kit — the drop-in pieces a content page composes. The upstream AI generation pipeline (how the prose and metadata are produced) is documented separately in the blog content engine spec.

---

## 1. Overview

The kit is a small set of single-purpose components under `components/geo/`. Each one solves a specific GEO/SEO job: it either renders an **extractable visual block** (something an AI can lift verbatim), emits **structured data** (`application/ld+json`) that engines parse for rich results and grounding, or both.

| Component | Renders | Emits JSON-LD | Why it helps GEO / SEO |
|---|---|---|---|
| `TldrBox` | Summary `<aside>` | — | Gives AI a pre-written, quotable one-paragraph answer. The first thing an Overview lifts. Also targeted by `Speakable`. |
| `FaqSection` | Accordion of Q&A | `FAQPage` | Each Q&A becomes a directly citable answer pair; powers FAQ rich results and "People also ask". |
| `KeyTakeaways` | Numbered list | — | Scannable, atomic claims an engine can extract as bullet citations. |
| `ComparisonTable` | Two-column table | — | Structured head-to-head data — exactly what AI quotes for "X vs Y" queries. |
| `BreadcrumbNav` | Breadcrumb trail | `BreadcrumbList` | Site hierarchy for crawlers; breadcrumb rich result; helps engines understand page context. |
| `RelatedContent` | Link grid | — | Internal linking surface — distributes authority and crawl depth across the content cluster. |
| `ContentCTA` | Conversion block | — | Converts the traffic the GEO work earns. |
| `AuthorBio` | Author `<aside>` | `Person` | E-E-A-T signal — attaches authorship/expertise the engines weight for trust. |
| `SpeakableSchema` | nothing (schema only) | `WebPage` + `SpeakableSpecification` | Marks the exact CSS selectors AI/voice assistants should read aloud or quote. |
| `HowToSchema` | nothing (schema only) | `HowTo` | Step rich results for procedural guides; steps become citable instructions. |
| `DefinedTermSchema` | nothing (schema only) | `DefinedTerm` | Glossary grounding — lets engines treat a page as the canonical definition of a term. |

Plus an **`Article` JSON-LD generator** composed inline on the page (see §4) that names the publisher, `about` topics, and `mentions` — the entity graph an AI uses to ground and attribute the page.

Design principles:

- **One component, one job.** Each file is tiny and independently portable.
- **Schema travels with the UI.** A component that shows FAQs also emits `FAQPage`, so you can never ship the visual without the structured data.
- **Pure props, no fetching.** Every component takes plain data props — they are framework-portable and trivially testable.

---

## 2. Page Composition

A content page is a thin shell: load metadata, render the body, and slot the kit around it. Below is the render skeleton (abstracted from a real `app/<type>/[slug]/page.tsx`).

```tsx
import { notFound } from "next/navigation";
import { getAllContent, getContentBySlug } from "@/lib/content";
import BreadcrumbNav from "@/components/geo/BreadcrumbNav";
import TldrBox from "@/components/geo/TldrBox";
import FaqSection from "@/components/geo/FaqSection";
import KeyTakeaways from "@/components/geo/KeyTakeaways";
import ContentCTA from "@/components/geo/ContentCTA";
import RelatedContent from "@/components/geo/RelatedContent";
import AuthorBio from "@/components/geo/AuthorBio";
import SpeakableSchema from "@/components/geo/SpeakableSchema";

const DEFAULT_AUTHOR = {
  name: "<YourApp>",
  bio: "<One-line description of who you are and what you help with>",
  url: "https://<yourapp>.com",
};

export default async function ContentPage({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;

  let meta;
  try {
    meta = getContentBySlug("guides", slug);
  } catch {
    notFound();
  }

  const MDXContent = MDX_COMPONENTS[slug]; // statically-imported MDX body
  if (!MDXContent) notFound();

  // Resolve related items from meta.relatedSlugs (fall back to recent posts)
  const allContent = getAllContent("guides");
  const relatedItems = (meta.relatedSlugs ?? [])
    .map((rs) => allContent.find((c) => c.slug === rs))
    .filter(Boolean)
    .map((c) => ({ href: `/guides/${c!.slug}`, title: c!.title, type: "Guide" }));

  // Article entity graph — see §4
  const articleLd = buildArticleLd(meta, slug);

  return (
    <article className="px-6 py-20">
      {/* 1. Page-level schema: Article + Speakable */}
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: JSON.stringify(articleLd) }}
      />
      <SpeakableSchema
        url={`https://<yourapp>.com/guides/${slug}`}
        cssSelectors={["h1", ".tldr-box", ".prose > p:first-of-type"]}
      />

      <div className="mx-auto max-w-2xl">
        {/* 2. Breadcrumb (emits BreadcrumbList) */}
        <BreadcrumbNav
          crumbs={[
            { label: "Home", href: "/" },
            { label: "Guides", href: "/guides" },
            { label: meta.title, href: `/guides/${slug}` },
          ]}
        />

        <h1 className="mt-4 text-3xl font-bold leading-tight md:text-4xl">{meta.title}</h1>

        {/* 3. TL;DR — the quotable summary, above the fold */}
        {meta.tldr && (
          <TldrBox>
            <p>{meta.tldr}</p>
          </TldrBox>
        )}

        {/* 4. The article body */}
        <div className="prose mt-12 max-w-none">
          <MDXContent />
        </div>

        {/* 5. FAQ (emits FAQPage) */}
        {meta.faqs?.length > 0 && <FaqSection faqs={meta.faqs} />}

        {/* 6. Key takeaways */}
        {meta.takeaways?.length > 0 && <KeyTakeaways items={meta.takeaways} />}

        {/* 7. Trust signal (emits Person) */}
        <AuthorBio author={DEFAULT_AUTHOR} />

        {/* 8. Conversion */}
        <ContentCTA />

        {/* 9. Internal linking */}
        {relatedItems.length > 0 && <RelatedContent items={relatedItems} />}
      </div>
    </article>
  );
}
```

**Composition order matters for GEO:** breadcrumb (context) → H1 → TL;DR (the extractable answer, early) → body → FAQ/takeaways (citable atoms) → author (trust) → CTA → related (linking). The TL;DR, H1, and first paragraph are exactly the selectors handed to `SpeakableSchema`.

All GEO content — `tldr`, `faqs`, `takeaways`, `relatedSlugs` — lives on the metadata object (`ContentMeta`), so the page shell stays generic and every component is conditionally rendered only when its data exists.

```ts
// lib/types.ts — the GEO fields the kit consumes
export interface ContentMeta {
  title: string;
  slug: string;
  excerpt: string;
  category: string;
  date: string;
  author?: string;
  readTime?: string;
  // GEO fields
  tldr?: string;
  faqs?: Array<{ question: string; answer: string }>;
  takeaways?: string[];
  relatedSlugs?: string[];
}
```

---

## 3. The Components

Real code follows. Brand color tokens (e.g. `ladya-red`) and palette hexes are kept verbatim — swap them for your own theme tokens when porting.

### 3.1 TL;DR Box

The single most important GEO block: a pre-written, quotable summary placed above the fold. Targeted by `Speakable` (`.tldr-box`). Renders `children` so the body can be rich.

```tsx
// components/geo/TldrBox.tsx
export default function TldrBox({ children }: { children: React.ReactNode }) {
  return (
    <aside className="my-8 rounded-lg border-l-4 border-ladya-red bg-[#faf6f0] p-6" aria-label="Summary">
      <p className="text-sm font-semibold uppercase tracking-wider text-ladya-red mb-2">TL;DR</p>
      <div className="text-[#1a1714] leading-relaxed">{children}</div>
    </aside>
  );
}
```

> Note: the `Speakable` selector in §2 references `.tldr-box`. Add `className="tldr-box"` to the `<aside>` (or wrap the box) when wiring the speakable selectors, so the schema target exists in the DOM.

### 3.2 FAQ Section (+ FAQPage JSON-LD)

Renders an accessible `<details>` accordion **and** emits `FAQPage` structured data from the same `faqs` array — the schema can never drift from the UI. Each Q&A becomes an independently citable answer.

```tsx
// components/geo/FaqSection.tsx
interface FaqItem {
  question: string;
  answer: string;
}

export default function FaqSection({ faqs }: { faqs: FaqItem[] }) {
  const jsonLd = {
    "@context": "https://schema.org",
    "@type": "FAQPage",
    mainEntity: faqs.map((faq) => ({
      "@type": "Question",
      name: faq.question,
      acceptedAnswer: { "@type": "Answer", text: faq.answer },
    })),
  };

  return (
    <section className="my-12">
      <script type="application/ld+json" dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd) }} />
      <h2 className="text-2xl font-bold text-[#1a1714] mb-6">Frequently Asked Questions</h2>
      <div className="space-y-3">
        {faqs.map((faq, i) => (
          <details key={i} className="group rounded-lg border border-[#e8e0d4] bg-white p-4">
            <summary className="cursor-pointer font-medium text-[#1a1714] list-none flex items-center justify-between">
              {faq.question}
              <span className="text-[#9a9188] group-open:rotate-180 transition-transform">&#x25BE;</span>
            </summary>
            <p className="mt-3 text-[#7a7268] leading-relaxed">{faq.answer}</p>
          </details>
        ))}
      </div>
    </section>
  );
}
```

### 3.3 Breadcrumb Nav (+ BreadcrumbList JSON-LD)

Renders a breadcrumb trail and emits `BreadcrumbList`. The last crumb is the current page (non-link, `aria-current="page"`). Absolute URLs in the schema are required — replace the hardcoded origin with your domain (or an env var) when porting.

```tsx
// components/geo/BreadcrumbNav.tsx
import Link from "next/link";

interface Crumb {
  label: string;
  href: string;
}

export default function BreadcrumbNav({ crumbs }: { crumbs: Crumb[] }) {
  const jsonLd = {
    "@context": "https://schema.org",
    "@type": "BreadcrumbList",
    itemListElement: crumbs.map((crumb, i) => ({
      "@type": "ListItem",
      position: i + 1,
      name: crumb.label,
      item: `https://<yourapp>.com${crumb.href}`,
    })),
  };

  return (
    <>
      <script type="application/ld+json" dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd) }} />
      <nav aria-label="Breadcrumb" className="text-sm text-[#9a9188]">
        <ol className="flex items-center gap-1.5">
          {crumbs.map((crumb, i) => (
            <li key={i} className="flex items-center gap-1.5">
              {i > 0 && <span>/</span>}
              {i === crumbs.length - 1 ? (
                <span className="text-[#1a1714] font-medium" aria-current="page">{crumb.label}</span>
              ) : (
                <Link href={crumb.href} className="hover:text-[#1a1714] transition-colors">{crumb.label}</Link>
              )}
            </li>
          ))}
        </ol>
      </nav>
    </>
  );
}
```

### 3.4 Key Takeaways

A numbered list of atomic claims. Each item is a self-contained, scannable statement an engine can lift as a bullet citation.

```tsx
// components/geo/KeyTakeaways.tsx
export default function KeyTakeaways({ items }: { items: string[] }) {
  return (
    <section className="my-12 rounded-lg bg-[#faf6f0] p-6">
      <h2 className="text-lg font-bold text-[#1a1714] mb-4">Key Takeaways</h2>
      <ol className="space-y-2">
        {items.map((item, i) => (
          <li key={i} className="flex gap-3 text-[#1a1714]">
            <span className="flex-shrink-0 w-6 h-6 rounded-full bg-ladya-red text-white text-sm flex items-center justify-center font-semibold">{i + 1}</span>
            <span>{item}</span>
          </li>
        ))}
      </ol>
    </section>
  );
}
```

### 3.5 Comparison Table

Structured head-to-head data for "X vs Y" queries — the format AI engines quote most reliably for comparisons. Optional `verdict` line gives the engine a ready-made conclusion. (Used on comparison/landing pages such as `app/for/<platform>/page.tsx`.)

```tsx
// components/geo/ComparisonTable.tsx
interface ComparisonRow {
  feature: string;
  left: string;
  right: string;
}

export default function ComparisonTable({
  leftLabel, rightLabel, rows, verdict,
}: {
  leftLabel: string;
  rightLabel: string;
  rows: ComparisonRow[];
  verdict?: string;
}) {
  return (
    <div className="my-8 overflow-x-auto">
      <table className="w-full border-collapse text-sm">
        <thead>
          <tr className="border-b-2 border-[#e8e0d4]">
            <th className="text-left p-3 text-[#9a9188] font-medium">Feature</th>
            <th className="text-left p-3 font-semibold text-[#1a1714]">{leftLabel}</th>
            <th className="text-left p-3 font-semibold text-ladya-red">{rightLabel}</th>
          </tr>
        </thead>
        <tbody>
          {rows.map((row, i) => (
            <tr key={i} className="border-b border-[#e8e0d4]">
              <td className="p-3 font-medium text-[#1a1714]">{row.feature}</td>
              <td className="p-3 text-[#7a7268]">{row.left}</td>
              <td className="p-3 text-[#1a1714]">{row.right}</td>
            </tr>
          ))}
        </tbody>
      </table>
      {verdict && (
        <p className="mt-4 text-sm font-medium text-ladya-red">{verdict}</p>
      )}
    </div>
  );
}
```

### 3.6 Related Content

Internal-linking grid that distributes crawl depth and authority across the content cluster. Each item carries a `type` label (e.g. "Guide", "Insight").

```tsx
// components/geo/RelatedContent.tsx
import Link from "next/link";

export default function RelatedContent({ items }: { items: Array<{ href: string; title: string; type: string }> }) {
  return (
    <section className="my-12">
      <h2 className="text-lg font-bold text-[#1a1714] mb-4">Related Reading</h2>
      <div className="grid gap-3 sm:grid-cols-2">
        {items.map((item) => (
          <Link
            key={item.href}
            href={item.href}
            className="rounded-lg border border-[#e8e0d4] p-4 hover:border-ladya-red/30 hover:bg-[#faf6f0] transition-colors"
          >
            <span className="text-xs font-semibold uppercase tracking-wider text-[#9a9188]">{item.type}</span>
            <p className="mt-1 font-medium text-[#1a1714]">{item.title}</p>
          </Link>
        ))}
      </div>
    </section>
  );
}
```

### 3.7 Content CTA

The conversion block that monetizes earned GEO traffic. Hardcoded copy/links — parameterize on port.

```tsx
// components/geo/ContentCTA.tsx
import Link from "next/link";

export default function ContentCTA() {
  return (
    <section className="my-12 rounded-2xl bg-gradient-to-br from-[#1a1714] to-[#2a2520] p-8 text-center">
      <h2 className="text-2xl font-bold text-white mb-3">Stop guessing. Start optimizing.</h2>
      <p className="text-[#c5beb4] mb-6 max-w-md mx-auto">
        <YourApp> <one-line value prop>.
      </p>
      <Link
        href="https://app.<yourapp>.com/signup"
        className="inline-flex items-center gap-2 rounded-full bg-ladya-red px-8 py-3 text-sm font-semibold text-white hover:bg-red-700 transition-colors"
      >
        Get Started for FREE
      </Link>
    </section>
  );
}
```

### 3.8 Speakable Schema (no UI)

Emits a `WebPage` with a `SpeakableSpecification` that points at the CSS selectors AI/voice assistants should read aloud or quote. Pair it with the selectors of your most quotable blocks — typically the `h1`, the TL;DR box, and the lead paragraph.

```tsx
// components/geo/SpeakableSchema.tsx
interface Props {
  url: string;
  cssSelectors: string[];
}

export default function SpeakableSchema({ url, cssSelectors }: Props) {
  const jsonLd = {
    "@context": "https://schema.org",
    "@type": "WebPage",
    url,
    speakable: {
      "@type": "SpeakableSpecification",
      cssSelector: cssSelectors,
    },
  };

  return (
    <script
      type="application/ld+json"
      dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd) }}
    />
  );
}
```

Usage (from §2):

```tsx
<SpeakableSchema
  url={`https://<yourapp>.com/guides/${slug}`}
  cssSelectors={["h1", ".tldr-box", ".prose > p:first-of-type"]}
/>
```

### 3.9 Bonus schema components (no UI)

Two more schema-only emitters for specialized page types. Drop them in when the content fits.

**HowTo** — for procedural guides; produces step rich results and citable instructions.

```tsx
// components/geo/HowToSchema.tsx
interface HowToStep {
  name: string;
  text: string;
}

interface Props {
  title: string;
  description: string;
  steps: HowToStep[];
}

export default function HowToSchema({ title, description, steps }: Props) {
  const jsonLd = {
    "@context": "https://schema.org",
    "@type": "HowTo",
    name: title,
    description,
    step: steps.map((step, i) => ({
      "@type": "HowToStep",
      position: i + 1,
      name: step.name,
      text: step.text,
    })),
  };

  return (
    <script type="application/ld+json" dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd) }} />
  );
}
```

**DefinedTerm** — for glossary / "what is X" pages; lets engines treat the page as the canonical definition of a term within a named term set.

```tsx
// components/geo/DefinedTermSchema.tsx
interface Props {
  term: string;
  definition: string;
  url: string;
}

export default function DefinedTermSchema({ term, definition, url }: Props) {
  const jsonLd = {
    "@context": "https://schema.org",
    "@type": "DefinedTerm",
    name: term,
    description: definition,
    url,
    inDefinedTermSet: {
      "@type": "DefinedTermSet",
      name: "<Your Glossary Name>",
      url: "https://<yourapp>.com/learn",
    },
  };

  return (
    <script type="application/ld+json" dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd) }} />
  );
}
```

---

## 4. Structured Data Generators

Two generators carry the heaviest GEO weight: the page-level **Article** entity graph and the **Author/Person** signal. The FAQ and Breadcrumb generators are co-located inside their components (§3.2, §3.3) so the schema and markup never diverge.

### 4.1 Article JSON-LD (page-level entity graph)

Built inline on the content page from the metadata object. Beyond the basics (`headline`, `datePublished`, `author`, `publisher`), the GEO-critical fields are `about` (topic entities), `mentions` (named entities the page references), and `inLanguage` — these tell an AI *what the page is about* and *which entities it grounds*, which is how it decides whether to cite you.

```ts
function buildArticleLd(meta: ContentMeta, slug: string) {
  return {
    "@context": "https://schema.org",
    "@type": "Article",
    headline: meta.title,
    description: meta.excerpt,
    datePublished: meta.date,
    url: `https://<yourapp>.com/guides/${slug}`,
    author: {
      "@type": "Organization",
      name: "<YourApp>",
      url: "https://<yourapp>.com",
      sameAs: [
        "https://instagram.com/<yourapp>",
        "https://linkedin.com/company/<yourapp>",
        "https://x.com/<yourapp>",
      ],
    },
    publisher: { "@type": "Organization", name: "<YourApp>", url: "https://<yourapp>.com" },
    mainEntityOfPage: { "@type": "WebPage", "@id": `https://<yourapp>.com/guides/${slug}` },
    about: [
      { "@type": "Thing", name: "<Primary topic>" },
      { "@type": "Thing", name: "<Secondary topic>" },
    ],
    mentions: [
      { "@type": "Organization", name: "<Entity you reference>" },
      { "@type": "Organization", name: "<Another entity>" },
    ],
    educationalLevel: "Professional",
    inLanguage: "en-IN",
  };
}
```

Rendered once near the top of the `<article>`:

```tsx
<script
  type="application/ld+json"
  dangerouslySetInnerHTML={{ __html: JSON.stringify(buildArticleLd(meta, slug)) }}
/>
```

### 4.2 Author / Person JSON-LD (E-E-A-T)

`AuthorBio` both renders the byline block and emits a `Person` — the authorship/expertise signal engines weight for trust. The `url` falls back to your homepage when no profile is provided.

```tsx
// components/geo/AuthorBio.tsx
import Link from "next/link";

interface AuthorInfo {
  name: string;
  bio: string;
  url?: string;
  image?: string;
}

export default function AuthorBio({ author }: { author: AuthorInfo }) {
  const jsonLd = {
    "@context": "https://schema.org",
    "@type": "Person",
    name: author.name,
    description: author.bio,
    url: author.url || "https://<yourapp>.com",
    ...(author.image && { image: author.image }),
  };

  return (
    <>
      <script type="application/ld+json" dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd) }} />
      <aside className="my-12 rounded-lg border border-[#e8e0d4] bg-[#faf6f0] p-6">
        <div className="flex items-start gap-4">
          {author.image && (
            <img src={author.image} alt={author.name} className="h-12 w-12 rounded-full object-cover flex-shrink-0" />
          )}
          <div className="flex-1">
            <p className="text-xs font-semibold uppercase tracking-wider text-[#9a9188] mb-1">Written by</p>
            <h3 className="font-semibold text-[#1a1714]">{author.name}</h3>
            <p className="mt-1 text-sm text-[#7a7268] leading-relaxed">{author.bio}</p>
            {author.url && (
              <Link
                href={author.url}
                className="mt-3 inline-block text-sm font-medium text-ladya-red hover:text-red-700 transition-colors"
                target="_blank"
                rel="noopener noreferrer"
              >
                View Profile →
              </Link>
            )}
          </div>
        </div>
      </aside>
    </>
  );
}
```

### Schema injection pattern

Every generator uses the identical injection pattern — a server-rendered `<script type="application/ld+json">` with stringified JSON. Because these render server-side, the schema is present in the initial HTML (no client hydration needed) so crawlers and AI fetchers see it immediately:

```tsx
<script type="application/ld+json" dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd) }} />
```

---

## 5. Dependencies

| Dependency | Used by | Notes |
|---|---|---|
| **React** (`React.ReactNode`, function components) | all | Core. Components are plain function components with typed props. |
| **Next.js** `next/link` | `BreadcrumbNav`, `RelatedContent`, `ContentCTA`, `AuthorBio` | Only for client-side nav. **Framework-agnostic swap:** replace `<Link href>` with a plain `<a href>` and these components work in any React stack (Remix, Vite, Astro islands, CRA). |
| **Next.js** server rendering | schema injection | The `<script type="application/ld+json">` pattern works in any SSR/SSG renderer. With pure CSR you must ensure the schema is in the served HTML (prerender) so crawlers see it. |
| **Tailwind CSS** (utility classes + theme tokens like `ladya-red`) | all visual components | Styling only. Swap utility classes / tokens for your own system; no logic depends on them. |
| **TypeScript** | all | Interfaces (`FaqItem`, `Crumb`, `ComparisonRow`, `ContentMeta`, …) are optional but recommended. |

The **schema-only** components (`SpeakableSchema`, `HowToSchema`, `DefinedTermSchema`) and the JSON-LD generators (§4) have **zero** runtime dependencies beyond React — they emit a `<script>` tag and nothing else, so they are the most portable pieces in the kit.

No external SEO library, no `next/head`, no third-party schema package is required.

---

## 6. Porting Checklist

- [ ] Copy `components/geo/` into your project (`TldrBox`, `FaqSection`, `KeyTakeaways`, `ComparisonTable`, `BreadcrumbNav`, `RelatedContent`, `ContentCTA`, `AuthorBio`, `SpeakableSchema`, plus optional `HowToSchema` / `DefinedTermSchema`).
- [ ] Replace the hardcoded origin `https://<yourapp>.com` in `BreadcrumbNav`, `SpeakableSchema` usage, `AuthorBio`, and the `Article` generator with your domain (prefer an env var / config constant).
- [ ] Swap brand tokens (`ladya-red`) and palette hexes (`#faf6f0`, `#1a1714`, `#e8e0d4`, `#9a9188`, `#7a7268`) for your theme.
- [ ] Rewrite `ContentCTA` copy and signup link.
- [ ] Set `DEFAULT_AUTHOR` (name, bio, url) and your `sameAs` social profiles in the `Article` generator.
- [ ] Fill `about` / `mentions` in the `Article` generator with your domain's topic and named entities.
- [ ] Add the GEO fields (`tldr`, `faqs`, `takeaways`, `relatedSlugs`) to your content metadata type/registry.
- [ ] If not using Next.js: replace `next/link` with `<a href>`.
- [ ] Ensure schema renders server-side / in the prerendered HTML (verify it appears in "View Source", not just the hydrated DOM).
- [ ] Add `className="tldr-box"` to the TL;DR box (or adjust the `Speakable` selectors) so the speakable target exists in the DOM.
- [ ] Compose components on your content page in GEO order: breadcrumb → H1 → TL;DR → body → FAQ → takeaways → author → CTA → related.
- [ ] Validate output with [Google Rich Results Test](https://search.google.com/test/rich-results) and [Schema.org validator](https://validator.schema.org/) — confirm `Article`, `FAQPage`, `BreadcrumbList`, `Person`, and `WebPage`/`speakable` all parse.

---

*Built by [Quantana](https://quantana.in). MIT licensed.*
