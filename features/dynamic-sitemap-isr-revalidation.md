# Dynamic Sitemap + On-Demand ISR Revalidation

> A database-driven `sitemap.xml` that always reflects your published content, paired with a secret-protected revalidate endpoint that refreshes individual static pages the moment content changes. After any publish or edit, your app fires a request to the endpoint, Next.js re-renders just the affected paths, and the sitemap picks up the new entry — no full rebuild, no stale cache.

## 1. Overview

Static pages in the Next.js App Router are fast because they are pre-rendered and cached. The downside is that they go stale when the underlying content changes. This feature solves both halves of that problem:

- **Dynamic sitemap** — `app/sitemap.ts` queries the database for all published content and emits a fresh `MetadataRoute.Sitemap` on every request, with per-entry `priority` and `changeFrequency`. New content appears in the sitemap automatically.
- **On-demand revalidation** — `app/api/revalidate/route.ts` exposes a `GET` endpoint guarded by a shared secret. When called with a `path`, it invokes Next.js `revalidatePath(path)`, evicting the cached render so the next visitor gets a freshly generated page.
- **Automatic triggering** — your content publisher calls the revalidate endpoint for the affected paths (e.g. the index page and the individual content page) immediately after writing to the DB.

The secret is the security boundary: without it the endpoint would let anyone force unlimited re-renders of arbitrary paths, a cheap denial-of-service / cache-stampede vector. The check rejects any request whose `secret` query param does not match `REVALIDATE_SECRET`.

## 2. Flow

```
  Content change (publish / edit / delete)
              │
              ▼
  publisher writes to DB  ──────────────►  blog_posts (status = "published")
              │
              ▼
  triggerRevalidation(slug)
   fetch /api/revalidate?path=/content&secret=***
   fetch /api/revalidate?path=/content/<slug>&secret=***
              │
              ▼
  /api/revalidate route
   ├── secret !== REVALIDATE_SECRET ──►  401 Invalid secret  (abuse blocked)
   ├── missing path                 ──►  400 Missing path
   └── revalidatePath(path)         ──►  { revalidated: true, path }
              │
              ▼
  Next.js evicts cached render for that path
              │
              ▼
  Next request to /content/<slug>  ──►  fresh page
  Next request to /sitemap.xml      ──►  fresh entry (re-queried from DB)
```

## 3. Dynamic Sitemap

`app/sitemap.ts` returns a `MetadataRoute.Sitemap`. It is marked `force-dynamic` so it is regenerated from the database on every request rather than baked in at build time. Each published row becomes an entry with its real `lastModified` timestamp, a `changeFrequency`, and a `priority`. The DB call is wrapped in a `try/catch` so the build still succeeds (returning the static entries only) if the database is unreachable during build.

```ts
// app/sitemap.ts
import { db } from "@/lib/db";
import { contentItems } from "@/lib/db/schema";
import { eq } from "drizzle-orm";
import type { MetadataRoute } from "next";

export const dynamic = "force-dynamic";

const SITE_URL = "https://yourapp.com";

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  let contentEntries: MetadataRoute.Sitemap = [];

  try {
    const items = await db
      .select({
        slug: contentItems.slug,
        updatedAt: contentItems.updatedAt,
      })
      .from(contentItems)
      .where(eq(contentItems.status, "published"));

    contentEntries = items.map((item) => ({
      url: `${SITE_URL}/content/${item.slug}`,
      lastModified: item.updatedAt,
      changeFrequency: "weekly" as const,
      priority: 0.7,
    }));
  } catch {
    // DB not available during build — return static entries only
  }

  return [
    {
      url: SITE_URL,
      lastModified: new Date(),
      changeFrequency: "daily",
      priority: 1,
    },
    {
      url: `${SITE_URL}/content`,
      lastModified: new Date(),
      changeFrequency: "daily",
      priority: 0.8,
    },
    ...contentEntries,
  ];
}
```

**Conventions used:**

| Entry            | priority | changeFrequency | Notes                                  |
| ---------------- | -------- | --------------- | -------------------------------------- |
| Homepage         | `1`      | `daily`         | Highest priority root URL              |
| Content index    | `0.8`    | `daily`         | Listing page, updated as items publish |
| Content item     | `0.7`    | `weekly`        | One per published row, real `lastModified` |

## 4. Revalidate Endpoint

`app/api/revalidate/route.ts` is a single `GET` handler. It reads `path` and `secret` from the query string, rejects mismatched secrets with `401`, rejects a missing `path` with `400`, and otherwise calls `revalidatePath(path)`.

```ts
// app/api/revalidate/route.ts
import { NextRequest, NextResponse } from "next/server";
import { revalidatePath } from "next/cache";

export async function GET(req: NextRequest) {
  const path = req.nextUrl.searchParams.get("path");
  const secret = req.nextUrl.searchParams.get("secret");

  if (secret !== (process.env.REVALIDATE_SECRET || "change-me")) {
    return NextResponse.json({ error: "Invalid secret" }, { status: 401 });
  }

  if (!path) {
    return NextResponse.json({ error: "Missing path" }, { status: 400 });
  }

  revalidatePath(path);
  return NextResponse.json({ revalidated: true, path });
}
```

**Security note:** the secret is what stops this endpoint from being an open re-render trigger. Anyone who can hit `/api/revalidate` without the secret could otherwise force Next.js to discard caches for arbitrary paths repeatedly, causing cache stampedes and wasted compute. Keep `REVALIDATE_SECRET` long, random, and out of source control; never ship the fallback default to production.

**Tag-based variant.** If you cache data with `fetch(..., { next: { tags: ["content"] } })` or `unstable_cache(..., { tags })`, you can invalidate by tag instead of by path. Swap `path`/`revalidatePath` for `tag`/`revalidateTag`:

```ts
import { revalidateTag } from "next/cache";

const tag = req.nextUrl.searchParams.get("tag");
// ...secret check...
if (tag) {
  revalidateTag(tag);
  return NextResponse.json({ revalidated: true, tag });
}
```

## 5. Triggering Revalidation

The publisher writes the content change to the database, then calls the revalidate endpoint for every path affected by that change. Here both the content index (`/content`) and the specific item page (`/content/<slug>`) are refreshed in parallel. The same secret used by the endpoint is read from the environment.

```ts
// lib/content/publisher.ts
import { db } from "@/lib/db";
import { contentItems } from "@/lib/db/schema";
import { eq, and, lte, asc } from "drizzle-orm";

export async function publishNextScheduledItem() {
  const now = new Date();

  const item = await db
    .select()
    .from(contentItems)
    .where(
      and(
        eq(contentItems.status, "scheduled"),
        lte(contentItems.scheduledAt, now)
      )
    )
    .orderBy(asc(contentItems.scheduledAt))
    .limit(1)
    .then((rows) => rows[0]);

  if (!item) return null;

  const [published] = await db
    .update(contentItems)
    .set({
      status: "published",
      publishedAt: now,
      updatedAt: now,
    })
    .where(eq(contentItems.id, item.id))
    .returning();

  await triggerRevalidation(published.slug);
  return published;
}

export async function triggerRevalidation(slug: string) {
  const baseUrl = process.env.APP_URL || "http://localhost:3000";
  const secret = process.env.REVALIDATE_SECRET || "change-me";
  await Promise.all([
    fetch(`${baseUrl}/api/revalidate?path=/content&secret=${secret}`),
    fetch(`${baseUrl}/api/revalidate?path=/content/${slug}&secret=${secret}`),
  ]);
}
```

Call `triggerRevalidation(slug)` from anywhere content mutates — scheduled publishing, manual edits, deletes (revalidate the index plus the now-gone path), or status changes. The sitemap needs no explicit trigger: because it is `force-dynamic`, it re-queries the DB on its next request automatically.

## 6. Dependencies

- **Next.js App Router** (`app/` directory). Uses `revalidatePath` / `revalidateTag` from `next/cache` and the `MetadataRoute.Sitemap` type — both App-Router-only APIs.
- **A database client** for reading published content. The examples use [Drizzle ORM](https://orm.drizzle.team/) (`db`, schema, `eq`/`and`/`lte`/`asc`), but any client works — the only requirement is a query for "all published items" with a `slug` and an `updatedAt`.
- **Environment variables:**

  | Variable             | Used by                         | Purpose                                                            |
  | -------------------- | ------------------------------- | ------------------------------------------------------------------ |
  | `REVALIDATE_SECRET`  | revalidate route + publisher    | Shared secret that authorizes revalidation. Must match on both sides. |
  | `APP_URL`            | publisher                       | Base URL the publisher fetches to reach `/api/revalidate`.         |
  | `DATABASE_URL`       | `db` client / sitemap / publisher | Connection string for the content database.                        |

- **A way to run the publisher** — a cron job, queue worker, webhook, or admin action that calls `publishNextScheduledItem()` / `triggerRevalidation()`.

## 7. Porting Checklist

- [ ] Copy `app/sitemap.ts`; replace `contentItems`/`blogPosts` with your table and `SITE_URL` with your domain.
- [ ] Confirm your published-content query returns `slug` and `updatedAt`; keep the `try/catch` so builds survive a DB outage.
- [ ] Tune `priority` and `changeFrequency` per entry type (homepage, index, items).
- [ ] Copy `app/api/revalidate/route.ts`; keep the `secret !== process.env.REVALIDATE_SECRET` guard.
- [ ] Set a strong, random `REVALIDATE_SECRET` in your environment (do not ship the fallback default).
- [ ] Decide path-based (`revalidatePath`) vs tag-based (`revalidateTag`) invalidation; add the tag branch if needed.
- [ ] Add `triggerRevalidation(slug)` calls after every content mutation (publish, edit, delete, status change).
- [ ] Set `APP_URL` so the publisher can reach the revalidate endpoint in each environment.
- [ ] Verify end to end: publish an item → hit `/sitemap.xml` (new entry present) → hit `/content/<slug>` (fresh render).
- [ ] Confirm a request to `/api/revalidate` with a wrong/absent secret returns `401`.

---

*Built by [Quantana](https://quantana.in). MIT licensed.*
