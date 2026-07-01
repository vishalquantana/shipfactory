# ShipFactory

A collection of production-tested feature specs, ready to hand to AI or developers for quick implementation.

## What is this?

ShipFactory is an open-source library of `.md` feature specifications. Each file describes a common product feature in enough detail — user flows, components, APIs, database schemas, integrations, and a porting checklist — that you can hand it to an AI coding assistant or a developer and get a working implementation fast.

These specs are extracted from real, shipped products. No theory — just battle-tested patterns.

## Who is this for?

- **Teams porting features across products** — grab a spec, hand it to your AI assistant, ship it
- **Solo founders & developers** — skip the design phase for common features and go straight to building
- **Anyone using AI-assisted development** — these specs are structured to work well as AI prompts

## How to use

1. Browse the [`/features`](./features) directory
2. Pick a feature spec
3. Feed it to your AI coding assistant (Claude, Cursor, Copilot, etc.) or use it as a reference for manual implementation
4. Adapt to your stack and ship

Each spec includes a **Porting Checklist** at the bottom — a step-by-step list of everything you need to implement.

## Feature Specs

Browse by category. Each link opens a self-contained spec with code, schema, and a porting checklist.

### Content, SEO & Growth

| Feature | What the file is |
|---------|------------------|
| [Automated Blog Content Engine](./features/automated-blog-content-engine.md) | A GEO-optimized, AI-driven content pipeline: discovers topics from live SERP data, generates an article + matching SVG illustration with an LLM, stores it as MDX with typed metadata, renders it with rich JSON-LD, then commits, indexes, and notifies — fully unattended |
| [Weekly Social Carousel Generator](./features/weekly-social-carousel-generator.md) | An agent-curated "this week in…" Instagram carousel: styled HTML rendered to PNG slides via Playwright, with a sourced-only (no hallucination) news brief, then auto-posted as one carousel |
| [SEO/GEO Component Kit](./features/seo-geo-component-kit.md) | Drop-in React components + JSON-LD generators that make content pages citable by AI Overviews/ChatGPT/Perplexity — TL;DR box, FAQ (+FAQPage), breadcrumbs (+BreadcrumbList), key takeaways, comparison table, related content, and Article/Speakable schema |
| [Google Indexing API Integration](./features/google-indexing-api.md) | Push freshly published URLs to Google for near-instant crawl via service-account JWT auth, plus Google/Bing sitemap pinging |
| [Dynamic Sitemap + ISR Revalidation](./features/dynamic-sitemap-isr-revalidation.md) | A DB-driven Next.js sitemap and a secret-protected `revalidatePath` endpoint so static pages refresh the moment content changes |
| [Next.js PageSpeed Optimization](./features/nextjs-pagespeed-optimization.md) | A recipe for a 99+ PageSpeed score — self-hosted fonts, AVIF/WebP images, CSS-only animations, and Nginx/Cloudflare caching, with measured Core Web Vitals targets |

### Auth, Onboarding & Billing

| Feature | What the file is |
|---------|------------------|
| [Passwordless OTP Email Auth](./features/otp-email-auth.md) | A two-step, passwordless 6-digit email OTP login flow with rate limiting and NextAuth session creation (email provider swappable) |
| [Waitlist Capture](./features/waitlist-capture.md) | An email/waitlist capture endpoint with honeypot + timing bot detection, IP rate limiting, dedupe, UTM tracking, and team notification |
| [Subscription Billing](./features/subscription-billing.md) | A recurring-billing lifecycle (trial → active → past_due → cancelled) with HMAC webhook verification and state-derived access gating — built on Cashfree, designed for provider swap (Stripe/Razorpay) |

### Data, Analytics & Automation

| Feature | What the file is |
|---------|------------------|
| [CSV Ingestion Pipeline](./features/csv-ingestion-pipeline.md) | Multi-source CSV onboarding — sniff the format from header signatures, map columns via a registry, coerce types, batch-insert with dedupe, and archive the raw file |
| [Pluggable Insights Engine](./features/insights-engine.md) | A framework that runs N modular analysis passes over your data and rolls them into an LLM executive summary — swap the domain passes, keep the engine |
| [AI Recommendation Engine](./features/ai-recommendation-engine.md) | Concurrent LLM analyzers produce scored, de-duplicated, actionable recommendations with an approve/reject/modify workflow and behavioral learning from user decisions |
| [Threshold Alert Monitoring](./features/threshold-alert-monitoring.md) | An aggregate → compare-to-configurable-threshold → severity-scored-alert pattern for real-time anomaly detection, with thresholds stored as JSONB |
| [Data Freshness Tracking](./features/data-freshness-tracking.md) | Per-source staleness status (fresh / stale / outdated by age thresholds) with a dashboard widget and an onboarding checklist — for any upload/sync/scraper SaaS |
| [Playwright Vendor-Portal Automation](./features/playwright-portal-automation.md) | A template for integrating a vendor that has a web portal but no API — OTP login, persisted browser session (`storageState`), report generation, table scraping, and concurrent-session management |

### Feedback & Support

| Feature | What the file is |
|---------|------------------|
| [Right-Click Bug Reporter](./features/rightclickbugreport.md) | In-app feedback & screenshot bug reporter with auto-capture, S3 upload, email notifications, and Jira integration |
| [Right-Click Bug Reporter (Plane)](./features/rightclickbugreport-plane.md) | The same in-app feedback & screenshot reporter, wired to Plane project management instead of Jira |

### AI, Chat & Documents

| Feature | What the file is |
|---------|------------------|
| [AI Chatbot with RAG](./features/chatbotrag.md) | Intent-routed AI chatbot with pgvector RAG pipeline, proposed-changes review UI, 3-layer deduplication, voice/file input, and meeting prep |
| [Chatbot Test & Self-Improvement Harness](./features/chatbot-test-harness.md) | A closed-loop harness for LLM chatbots — an LLM plays the user to simulate hundreds of full conversations, scores every transcript, visualizes each chat as a live tree, then runs a simulate → score → analyze → fix → re-test loop until the bot passes a stability bar |
| [Voice Conversational Intake](./features/voice-conversational-intake-prd.md) | Voice-powered form intake using ElevenLabs Conversational AI — full WebSocket protocol, agent setup, state machine, and client/server implementation |
| [Meeting Transcript Intelligence & RAG](./features/meeting-transcripts-rag.md) | End-to-end meeting-transcript pipeline — ingest from recorder bot / Google Meet / paste / file / API, LLM extraction into actions, notes, contacts & deal signals (auto-apply vs review), pgvector RAG over the whole collection, and a cited "Ask anything" agentic chat |
| [E-Signature Documents (Self-Hosted DocuSign)](./features/esignature-docusign.md) | DocuSign-style e-signature module — PDF field builder, ordered token-based signers, type/draw/upload signatures, server flatten + Certificate of Completion, tamper-evident hash-chained audit trail, plus client-side PDF compress & merge |

### Engineering & Agent Orchestration

| Feature | What the file is |
|---------|------------------|
| [Parallel Multi-Agent Dev System (cmux)](./features/orchestrator-cmux-multiagent.md) | An orchestrator supervising a fleet of coding agents (Claude/Codex/Gemini) across parallel cmux panes — each in an isolated git worktree on a `feat/*` branch, integrated by a single-writer merge-train (theirs-wins + post-merge integrity gate + one version stamp) and shipped by hands-off autodeploy. Merge conflicts are designed out, not resolved. Includes every script, the git hooks, the orchestrator runbook, and hard-won best practices |

## Spec Structure

Every feature spec follows a consistent format:

- **Overview** — what the feature does
- **User Flow** — step-by-step interaction diagram
- **Frontend Components** — component breakdown with props, triggers, and behavior
- **Frontend Dependencies** — packages and versions
- **Backend API** — endpoints, auth, request/response formats
- **Database Schema** — table definitions with constraints
- **Integrations** — third-party services (S3, email, issue trackers, etc.)
- **Data Collected** — what's captured automatically vs. manually
- **Porting Checklist** — actionable implementation steps

## Contributing

Have a feature you've shipped and want to share? PRs are welcome.

1. Add your spec to `/features` as a `.md` file
2. Follow the structure above (adapt sections as needed — not every feature needs all of them)
3. Include a porting checklist at the end
4. Update the feature table in this README

## About Us

We are [Quantana](https://quantana.com.au), an AI-first design and development agency working with Fortune 500s to build bespoke AI solutions and provide the audit and training needed to ensure success. [Click here to learn more](https://quantana.com.au).

## License

MIT
