# Next.js PageSpeed Optimization Recipe

> A concrete, copy-adaptable recipe for pushing a Next.js (App Router) marketing site to a 99+ PageSpeed Insights score. It covers self-hosted local fonts, `next/image` with AVIF/WebP, CSS-only scroll animations, and Nginx + Cloudflare caching/compression — every Core Web Vitals lever in one place. This is a guide, not a drop-in package: the code is real and battle-tested, but you adapt paths, domains, and config to `<YourApp>`.

---

## 1. Overview

This recipe targets the three Core Web Vitals that most often tank a marketing-site score:

- **LCP (Largest Contentful Paint)** — usually the above-the-fold `<h1>`. The enemy is render-blocking webfonts: a round-trip to `fonts.googleapis.com` + `fonts.gstatic.com` delays text paint. Fix: self-host fonts with `display: "swap"` so text renders on the first frame.
- **CLS (Cumulative Layout Shift)** — caused by images without explicit dimensions and by fonts swapping in with different metrics. Fix: `next/image` with `width`/`height`, and `font-display: swap` on local fonts.
- **Render-blocking / TBT (Total Blocking Time)** — caused by shipping JS for things that don't need it. The biggest offender on a content site is a JS animation library (or an `IntersectionObserver` on every section). Fix: move scroll animations to pure CSS (`animation-timeline: view()`), force static generation, and strip dead dependencies (analytics SDKs, empty middleware).

The remaining wins are delivery-layer: gzip/Brotli compression and aggressive `Cache-Control` headers at Nginx, plus a Cloudflare edge cache in front. None of these touch your design — they only change how bytes are produced and shipped.

Tech assumed: **Next.js 16 (App Router), Tailwind, Nginx reverse-proxy, Cloudflare free tier.** Most of it applies to any recent Next.js version.

---

## 2. Self-Hosted Fonts

Loading fonts from Google Fonts adds two external origins and a render-blocking dependency in the critical path. `next/font/local` inlines the `@font-face` CSS, self-hosts the WOFF2, and auto-injects a preload — eliminating the round-trips entirely.

**Step 1 — download the WOFF2 subsets** into `public/fonts/`. To find the current URL for a font, inspect the Google Fonts CSS (e.g. `https://fonts.googleapis.com/css2?family=Inter:wght@100..900&display=swap`) and copy the `latin` block's `woff2` URL:

```bash
mkdir -p public/fonts
curl -o public/fonts/inter-latin.woff2 \
  "https://fonts.gstatic.com/s/inter/v18/UcCo3FwrK3iLTcviYwY.woff2"
curl -o public/fonts/parisienne-latin.woff2 \
  "https://fonts.gstatic.com/s/parisienne/v13/E21i_d3kvfOmhHEt0Hnh.woff2"
```

**Step 2 — swap `next/font/google` for `next/font/local`** in `app/layout.tsx`.

Before:

```ts
import { Inter, Parisienne } from "next/font/google";

const inter = Inter({
  subsets: ["latin"],
  variable: "--font-sans",
  display: "swap",
});

const parisienne = Parisienne({
  weight: "400",
  subsets: ["latin"],
  variable: "--font-parisienne",
  display: "swap",
});
```

After (real code from `<YourApp>`'s `app/layout.tsx`):

```ts
import localFont from "next/font/local";

const inter = localFont({
  src: "../public/fonts/inter-latin.woff2",
  variable: "--font-sans",
  display: "swap",
  weight: "100 900", // variable font weight range
});

const parisienne = localFont({
  src: "../public/fonts/parisienne-latin.woff2",
  variable: "--font-parisienne",
  display: "swap",
  weight: "400",
});
```

Apply the CSS variables on `<html>` and Tailwind/your CSS picks them up via `font-sans`:

```tsx
<html lang="en" className={`${inter.variable} ${parisienne.variable}`}>
  <body className="font-sans antialiased">
```

**Why this works:** `next/font/local` automatically emits the `<link rel="preload">` for you — do **not** add a manual preload, that causes a duplicate. `display: "swap"` means text paints immediately in a fallback face and swaps when the WOFF2 lands, so a text LCP element is never blocked. The `fonts.googleapis.com` and `fonts.gstatic.com` origins disappear from the network waterfall entirely.

---

## 3. Image Optimization

A single 1.1MB PNG hero image can dominate transfer size. `next/image` serves modern formats (AVIF ~50–80KB, WebP ~80–120KB for the same image), picks the right size per viewport, and enforces explicit dimensions so the image reserves space (no CLS).

**Step 1 — enable modern formats** in `next.config.ts`. Real config from `<YourApp>`:

```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  pageExtensions: ["js", "jsx", "md", "mdx", "ts", "tsx"],
  images: {
    formats: ["image/avif", "image/webp"], // AVIF preferred, WebP fallback
  },
  experimental: {
    optimizeCss: true, // inline + minify critical CSS
  },
};

export default nextConfig;
```

`formats` is ordered by preference: Next.js serves AVIF to browsers that accept it, falls back to WebP, then the original.

**Step 2 — replace raw `<img>` with `next/image`.**

Before:

```tsx
{/* eslint-disable-next-line @next/next/no-img-element */}
<img
  src="/hero.png"
  alt="<YourApp> — product hero"
  width={400}
  height={500}
  className="h-[420px] w-auto drop-shadow-lg rounded-xl"
  loading="lazy"
/>
```

After:

```tsx
import Image from "next/image";

<Image
  src="/hero.png"
  alt="<YourApp> — product hero"
  width={400}
  height={500}
  className="h-[420px] w-auto drop-shadow-lg rounded-xl"
  loading="lazy"
  sizes="(max-width: 768px) 90vw, 400px"
/>
```

**Key decisions:**
- `width` + `height` are mandatory — they set the aspect ratio so the browser reserves space → **zero CLS**.
- `sizes` tells Next.js which resolution to generate per breakpoint, so mobile never downloads the desktop-sized image.
- `loading="lazy"` is correct **only for below-the-fold images**. If the image *is* your LCP element, drop `loading="lazy"` and add `priority` instead. In `<YourApp>` the LCP is a text `<h1>`, so the hero image stays lazy.

---

## 4. CSS-Only Animations

JS scroll animations (animation libraries, or a hand-rolled `IntersectionObserver` wrapping every section) ship a client bundle and run on the main thread — pure TBT cost. Modern CSS does scroll-driven animation natively with `animation-timeline: view()`, which runs off the main thread at zero JS cost.

**Step 1 — make the wrapper a pure server component.** Real code from `<YourApp>`'s `components/ui/FadeUp.tsx` — note there is no `"use client"`, no `useEffect`, no `useRef`, no observer:

```tsx
interface FadeUpProps {
  children: React.ReactNode;
  className?: string;
  delay?: number;
}

export default function FadeUp({ children, className = "", delay = 0 }: FadeUpProps) {
  return (
    <div
      className={`fade-up ${className}`}
      style={delay ? { animationDelay: `${delay}ms` } : undefined}
    >
      {children}
    </div>
  );
}
```

This component now adds *only* a CSS class. It can be used freely inside server components and ships **zero JS**.

**Step 2 — drive the animation entirely in CSS** (in `globals.css`):

```css
/* Fade-up on scroll — pure CSS, no JS needed */
@keyframes fade-up {
  from { opacity: 0; transform: translateY(32px); }
  to   { opacity: 1; transform: translateY(0); }
}

.fade-up {
  animation: fade-up 0.7s cubic-bezier(0.16, 1, 0.3, 1) both;
  animation-timeline: view();          /* tie progress to scroll position */
  animation-range: entry 0% entry 40%; /* play as element enters viewport */
}
```

**Compatibility note (be honest about this):** `animation-timeline: view()` is supported in Chrome/Edge 115+ and Firefox 110+. In Safari < 17.5 it's ignored — the animation simply doesn't run, and content is visible immediately. That's a clean graceful fallback (no broken layout, no hidden content), which is exactly what you want versus a JS approach that risks hiding content if the observer never fires. For a reduced-motion-respecting variant, wrap the animation in `@media (prefers-reduced-motion: no-preference)`.

---

## 5. Caching & Compression

The app code is fast now; the last mile is shipping fewer bytes and caching them hard. Two layers: Nginx (origin) and Cloudflare (edge).

### Nginx — gzip + Cache-Control

Real config pattern from `<YourApp>`'s Nginx site file. Adjust `proxy_pass` port and cert paths:

```nginx
server {
    server_name <yourapp>.com;

    # --- Compression ---
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_min_length 256;
    gzip_types
        text/plain
        text/css
        text/javascript
        application/javascript
        application/json
        application/xml
        image/svg+xml
        font/woff2;

    # --- Next.js hashed static assets: immutable, cache forever ---
    location /_next/static/ {
        proxy_pass http://localhost:3001;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        add_header Cache-Control "public, max-age=31536000, immutable";
    }

    # --- Fonts & images: cache 7 days ---
    location ~* \.(woff2|woff|ttf|otf|png|jpg|jpeg|webp|avif|svg|ico)$ {
        proxy_pass http://localhost:3001;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        add_header Cache-Control "public, max-age=604800";
    }

    # --- HTML pages: always revalidate (let the CDN hold the cache) ---
    location / {
        proxy_pass http://localhost:3001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        add_header Cache-Control "public, max-age=0, must-revalidate";
    }

    listen 443 ssl; # managed by Certbot
    # ssl_certificate / ssl_certificate_key — your Let's Encrypt paths
}
```

The `immutable` on `/_next/static/*` is safe because Next.js content-hashes those filenames — a new build produces new URLs, so the year-long cache is never stale.

Test and reload, then verify:

```bash
nginx -t && systemctl reload nginx
curl -I https://<yourapp>.com/ | grep -i cache-control
# → Cache-Control: public, max-age=0, must-revalidate
```

> For Brotli (`gzip`'s smaller cousin), compile `ngx_brotli` and add `brotli on;` + `brotli_types ...`. If that's too much ops work, let **Cloudflare apply Brotli at the edge** instead (below) — same benefit, no origin module.

### Cloudflare (free tier) — edge cache + Brotli

1. Add the site, select **Free plan**, point your registrar's nameservers at Cloudflare.
2. DNS: `A` record for the app → origin IP, **Proxied** (orange cloud ON).
3. **SSL/TLS:** mode **Full (Strict)** (you have Let's Encrypt on origin); enable *Always Use HTTPS*; min TLS 1.2.
4. **Speed → Optimization:** Auto Minify HTML/CSS/JS; **Brotli: ON**.
5. **Caching:** Level *Standard*; Browser Cache TTL *Respect Existing Headers* (honors the Nginx headers above).
6. **Page Rules** (3 free):
   - `<yourapp>.com/_next/static/*` → Cache Everything, Edge Cache TTL 30 days
   - `<yourapp>.com/*.woff2` → Cache Everything, Edge Cache TTL 30 days

Verify the edge is live:

```bash
curl -I https://<yourapp>.com/ | grep -i cf-
# → cf-ray: ...   cf-cache-status: HIT
```

### Force static generation

So Nginx/Cloudflare serve pre-built HTML with zero per-request server compute, mark content pages static. In each route's `page.tsx`, after imports:

```ts
export const dynamic = "force-static";
```

For dynamic blog routes, use `generateStaticParams()` to pre-render every slug at build time. A correct build shows `○` (static) / `●` (SSG) markers — no `λ`/`ƒ` (server-rendered) on content pages.

---

## 6. Measured Impact

The spec this recipe is distilled from set these **success criteria** (and the implementation hit them):

| Metric | Target |
| --- | --- |
| PageSpeed Insights — desktop | **99+** |
| PageSpeed Insights — mobile | **95+** floor, **99+** once Cloudflare edge cache is warm |
| LCP (Largest Contentful Paint) | **< 1.2s** |
| CLS (Cumulative Layout Shift) | **< 0.05** |
| TBT (Total Blocking Time) | **< 50ms** |
| Render-blocking resources | **none** |

Where the gains come from:

- **Hero image:** ~**1.1MB PNG → ~80KB** AVIF/WebP via `next/image` (roughly a 90%+ transfer reduction on that asset).
- **Fonts:** two external origins (`fonts.googleapis.com`, `fonts.gstatic.com`) eliminated → no render-blocking font round-trip, text LCP paints on first frame.
- **JS:** removing the `IntersectionObserver` `FadeUp` (now a server component), a dead analytics SDK, and an empty `middleware.ts` cuts client JS and Edge Runtime overhead → TBT toward < 50ms.
- **Delivery:** gzip + immutable `Cache-Control` + Cloudflare edge cache → repeat visits and `_next/static` assets served from cache, near-instant.

Your absolute numbers depend on theme weight and third-party scripts, but the *direction* is reliable: each lever here removes a specific, named Core Web Vitals bottleneck.

---

## 7. Porting Checklist

- [ ] Download font WOFF2 subsets into `public/fonts/`; switch `next/font/google` → `next/font/local` with `display: "swap"` (no manual preload).
- [ ] Apply font CSS variables on `<html>`; confirm `fonts.gstatic.com` is gone from the network waterfall.
- [ ] Add `images: { formats: ["image/avif", "image/webp"] }` to `next.config.ts`.
- [ ] Replace raw `<img>` with `next/image` everywhere; set `width`/`height` (CLS) and `sizes`; use `priority` for the LCP image, `loading="lazy"` for below-fold.
- [ ] Convert scroll-animation wrappers to pure server components (drop `"use client"`/observers); drive with CSS `animation-timeline: view()`; verify graceful fallback in Safari.
- [ ] Remove dead deps (analytics SDKs you don't use) and any empty `middleware.ts`.
- [ ] Add `export const dynamic = "force-static"` to content pages; `generateStaticParams()` for dynamic routes; confirm `○`/`●` in build output.
- [ ] Configure Nginx: `gzip on` + `gzip_types`; `immutable` cache on `/_next/static/`; 7-day cache on fonts/images; `must-revalidate` on HTML. `nginx -t && systemctl reload nginx`.
- [ ] Put Cloudflare in front (Full Strict SSL, Brotli ON, Auto Minify, page rules for `_next/static` + `*.woff2`); verify `cf-cache-status`.
- [ ] Run PageSpeed Insights (mobile + desktop) and confirm LCP < 1.2s, CLS < 0.05, TBT < 50ms, no render-blocking resources.

---

*Built by [Quantana](https://quantana.in). MIT licensed.*
