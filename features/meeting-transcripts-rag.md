# Meeting Transcript Intelligence & RAG

## Feature Specification for Porting

This document describes an end-to-end **meeting-transcript intelligence** system: ingest meeting transcripts from multiple sources (meeting-recorder bot, Google Meet, manual paste, file upload, API/MCP), use an LLM to extract structured CRM data (action items, notes, contacts, deal signals), store everything as a searchable collection, embed it into a **pgvector RAG layer**, and expose a conversational **"Ask anything"** chat that answers grounded, cited questions across all of a user's meetings.

The reference implementation is a sales CRM (FastAPI/Python backend, Postgres + `pgvector`, React frontend, Google Gemini for LLM + embeddings). The patterns port to any domain where you turn unstructured conversation into structured records you can later query.

---

## 1. Overview

Four stages, each independently portable:

1. **Ingest** — get raw transcript text into the system from any of five sources, linked to a record (a "deal") and a meeting.
2. **Extract** — one LLM call turns the transcript into structured JSON: summary, notes, action items, events, contacts (with roles), and field updates. Low-risk items auto-apply; high-risk items go to a review queue.
3. **Collect** — every transcript is stored verbatim alongside its AI analysis, forming a per-record meeting history.
4. **Retrieve & Chat** — transcripts (and notes, emails, deals, contacts, docs) are chunked and embedded into a vector store. A conversational agent answers questions by running semantic search + structured fetch, citing its sources.

```
                 ┌──────────── INGEST ────────────┐
  Recorder bot ──┤                                 │
  Google Meet  ──┤   run_transcript_analysis()     │
  Manual paste ──┤   (single entry point for all)  │──┐
  File upload  ──┤                                 │  │
  MCP / API    ──┘                                 │  │
                 └─────────────────────────────────┘  │
                                                       ▼
            ┌──────────── EXTRACT (1 LLM call) ───────────────┐
            │  summary · notes · actions · events · contacts  │
            │  (MEDDIC roles) · field updates · sentiment      │
            │       ┌─────────────┬──────────────────┐         │
            │       │ auto-apply  │  review queue     │         │
            │       └─────────────┴──────────────────┘         │
            └──────────────────────┬──────────────────────────┘
                                   ▼
        ┌──── COLLECT ────┐   ┌──── EMBED (async) ────┐
        │ transcript_     │──▶│ chunk → embed → upsert │
        │ analyses (raw   │   │ embeddings (vector 768)│
        │ + AI response)  │   └───────────┬────────────┘
        └─────────────────┘               ▼
                              ┌──── CHAT / "Ask anything" ────┐
                              │  semantic search + spine fetch │
                              │  → grounded answer + citations │
                              │  → optional proposed action    │
                              └────────────────────────────────┘
```

> **Naming note:** in the source product the central record is a sales **deal** and transcripts are tied to deals. When porting, "deal" = whatever record a meeting is about (a project, a case, an account). The transcript table is `transcript_analyses`.

---

## 2. Ingestion Paths

All five paths converge on a single function, `run_transcript_analysis(transcript_text, workspace_id, user_id, deal_id?, source)`, so extraction behavior is identical regardless of source. Only how the raw text arrives differs.

### 2.1 Meeting-recorder bot (Recall.ai)

A bot joins the live meeting (Zoom / Google Meet / Teams), records, and streams a diarized transcript back via webhook.

| Step | Detail |
|------|--------|
| **Dispatch** | `POST /api/recall/bots` `{ meeting_url, deal_id?, calendar_event_id?, bot_name? }` → calls the Recall API `create_bot()` with a webhook subscription; inserts a `recall_bots` row + a placeholder `transcript_analyses` row |
| **Live stream** | Webhook `POST /api/webhooks/recall` event `transcript.data` → appends `{ speaker, words[] }` to a live table |
| **Finalize** | Webhook events `bot.done` / `bot.call_ended` atomically claim finalization (race-safe), fetch the final diarized transcript from Recall's signed S3 URL, normalize + format to `[HH:MM] Speaker: text`, update `transcript_analyses.transcript_text`, then call `run_transcript_analysis(source='recall')` |
| **Media offload** | Audio/video archived to your own S3 (`audio_s3_key`, `video_s3_key`); original media can later be deleted from the recorder to cut retention cost |
| **Webhook security** | Verify the webhook signature (Svix) with a shared secret before processing |
| **Env** | `RECALL_API_KEY`, `RECALL_REGION_BASE_URL` (default `https://us-east-1.recall.ai`), `RECALL_WEBHOOK_SECRET`, `RECALL_DEFAULT_BOT_NAME` |

Files: `routers/recall.py`, `routers/webhooks.py`, `services/recall.py`.

### 2.2 Google Meet transcript sync (polling)

For meetings recorded natively by Google Meet (Gemini notes / Meet transcripts), a background poller pulls finished transcripts from the Google Meet API and queues them for the user to confirm.

| Step | Detail |
|------|--------|
| **Poll** | Every 5 min per opted-in user: refresh Google OAuth token, list recent conference records (last 24h), skip ones already handled or covered by a recorder bot |
| **Fetch** | Pull participants + transcript entries + calendar event title for each conference |
| **Auto-match deal** | Match by **attendee emails → known contacts** and **title keyword overlap**; insert a `meet_transcript_syncs` row (`status='pending'`) with `matched_deals` + `participant_emails` |
| **User confirm** | `GET /api/meet-transcripts/pending` lists the queue; `POST /api/meet-transcripts/{id}/ingest` `{ deal_id?, edited_transcript? }` runs analysis (`source='meet'`); `POST /api/meet-transcripts/{id}/skip` dismisses |
| **Deal surfacing** | `GET /api/deals/{deal_id}/pending-transcripts` shows transcripts whose participants overlap the deal's contacts |

Files: `services/meet_transcripts.py`, `routers/transcript.py`.

### 2.3 Manual paste

`POST /api/analyze-transcript` `{ text, deal_id? }` → directly calls `run_transcript_analysis(source='manual')`. The manual path **bypasses the corroboration gate** (§4.4) — the user explicitly chose the deal, so extracted changes auto-apply to it.

### 2.4 File upload

`POST /api/analyze-transcript-file` (multipart, max 20 MB). Accepts `pdf, txt, md, vtt, srt, doc, docx`. DOCX/DOC are converted to text; other files are uploaded to the **Gemini Files API** and attached to the prompt as a file part. The file is also saved to S3 and recorded as an engagement on the deal. Then the standard pipeline runs.

### 2.5 API / MCP (`submit_transcript`)

An MCP tool (and equivalent REST surface) lets an external assistant submit a transcript programmatically:

```
submit_transcript(api_key, transcript_text, deal_name | deal_id,
                  meeting_date?, participants?)
```

Resolves the API key → `{ workspace_id, user_id }`, resolves the deal **with per-user visibility enforcement**, prepends a metadata header (date + participants), runs analysis (`source='mcp'`), and writes an audit-log entry. Returns a human-readable confirmation.

### 2.6 Email ingestion (extension — not in reference build)

> The reference implementation does **not** ingest transcripts from inbound email. If you want it: add an inbound-email webhook (e.g. SendGrid Inbound Parse / Mailgun Routes) that (a) verifies the sender, (b) extracts the transcript from the body or an attached `.txt/.vtt/.docx`, (c) matches a deal by the sender/participant domains, and (d) calls the same `run_transcript_analysis(source='email')`. Everything downstream is reused unchanged — this is purely a new front door.

### 2.7 Meeting ↔ record linkage

A transcript links to a deal via `transcript_analyses.deal_id` (nullable). It is set at dispatch (bot), at confirm (Meet), at upload (manual/file), or as a required arg (MCP). Participant emails and calendar titles drive auto-matching; unmatched transcripts still analyze (producing orphan actions + a meeting summary) and can be linked later.

---

## 3. The Extraction Pipeline

Central function: `run_transcript_analysis()`. It is the heart of the system — every ingestion path calls it.

### 3.1 Steps

1. **Assemble context** — fetch the workspace's deals, team members, existing contacts, and pipelines; if `deal_id` is set, also fetch that deal's current state, known contacts, the last few transcript summaries, and any PRD/scope excerpt.
2. **Build the prompt** — embed that context as JSON plus extraction instructions (see §3.2). A **deal-relevance check** explicitly forbids matching a meeting to a deal just because a name/email overlapped — it must match on real business context.
3. **Call the LLM** — model `gemini-2.5-flash-lite`, **dynamic thinking enabled** for this caller (reasoning matters for role/signal extraction), 290 s timeout, 2 retries on transient errors, quota-checked, token usage tracked per caller.
4. **Classify** changes into auto-apply vs review tiers (§4.3).
5. **Persist** the analysis (`transcript_analyses`: raw text + full LLM JSON + source + title).
6. **Queue embeddings** for the AI summary (`transcript`) and the verbatim text (`transcript_raw`), stamped with `deal_id`.
7. **Auto-apply** low-risk changes inside a savepoint (rollback-safe).
8. **Notify** — Slack DM + in-app notification summarizing what was created.

### 3.2 Output schema (LLM → JSON)

```json
{
  "meeting_summary": "string",
  "orphan_actions": [{ "title": "", "owner_name": "", "deadline": "" }],
  "deals": [{
    "deal_id": "uuid | null",
    "deal_name": "string",
    "is_new": false,
    "proposed_stage": "stage_slug | null",
    "proposed_amount": 0,
    "suggested_pipeline": "name | null",
    "proposed_changes": {
      "summary": "string",
      "notes": ["string"],
      "actions": [{ "title": "", "owner_name": "", "deadline": "" }],
      "events": [{ "type": "", "title": "", "event_date": "", "auto_actions": [] }],
      "field_updates": { "dealstage": "", "amount": 0, "closedate": "" },
      "sentiment": { "signals": ["string"], "risk_level": "low|medium|high" },
      "contacts": [{
        "name": "", "email": "", "title": "", "company": "",
        "role": "champion|economic_buyer|technical_buyer|coach|blocker|end_user|influencer",
        "confidence": "high|low",
        "existing_contact_id": "uuid | null",
        "personality": "string",
        "personal_intel": {
          "hobbies": [], "family": "string|null",
          "kids": [{ "name": "", "birth_month": 0, "birth_day": 0, "birth_year": 0 }],
          "key_events": [{ "event_type": "", "label": "", "month": 0, "day": 0 }],
          "other_notes": []
        }
      }]
    }
  }]
}
```

The extraction is **holistic** — one call captures everything (deals, MEDDIC roles, events, orphan actions, personal intel), not just a single artifact. Personal intel is only recorded when explicitly stated; the prompt forbids inventing facts.

---

## 4. What Gets Created (auto-apply vs review)

### 4.1 Records written on auto-apply

| Extracted item | Table(s) | Notes |
|----------------|----------|-------|
| Notes | `engagements` (type `note`) + `associations` link to deal | AI summary prefixed; optional contact link via `contact_notes` |
| Action items | `actions` | owner resolved to a member, deadline parsed, `status='open'`, `source_transcript_analysis_id` set; idempotency check avoids duplicate open actions; `post_closure=true` if the deal is already closed |
| Orphan actions | `actions` (`deal_id=NULL`) | internal tasks with no deal |
| Events | `events` (`status='upcoming'`) | spawn auto-actions (inherit/precede the event date) |
| Contacts | `contacts` + `deal_contacts` (with MEDDIC `role`) | created or enriched non-destructively |
| Contact enrichment | `contact_profiles` (personality, personal_notes, kids), `contact_key_events` (birthdays/anniversaries, deduped) | from `personal_intel` |

### 4.2 Records sent to review

New deals (`is_new`), field updates (`dealstage` / `amount` / `closedate`), and ambiguous contacts (single-token names or low LLM confidence). These are returned in `review_required` and applied only on user confirmation.

### 4.3 The classification rule

- **Auto** — notes, actions, events, sentiment, and **safe** contacts (full name + high confidence, OR matched to an existing contact).
- **Review** — anything that mutates a deal's core fields or creates a new deal, plus ambiguous contacts.
- **Reject (dropped silently)** — empty names and pure role placeholders ("Procurement Person", "Head Chef") via a generic-role-word filter.

### 4.4 Corroboration gate (auto-pipeline safety)

For *automatic* pipelines (Meet sync), the LLM can over-eagerly match a deal from content alone (e.g. a case study mentioned in passing). The gate only auto-applies to a deal that is **corroborated** by attendees (≥2 customer-side contacts matched) or title overlap. Deals matched purely from transcript content are demoted to the review tier. Manual paste bypasses the gate (the user already chose the deal).

### 4.5 Junk-contact filtering

Three tiers (`auto` / `review` / `reject`) plus an **internal-member guard**: extracted names are token-matched against workspace members so teammates are never created as customer contacts. Email-unverified, single-token, and generic-role contacts are filtered out of auto-apply. Metrics (`contacts_created`, `contacts_linked`, `contacts_skipped`) are logged per run.

---

## 5. The Transcript Collection (storage)

Every analyzed transcript is stored verbatim alongside its AI output, forming a per-deal meeting history that the detail page and RAG both read from.

```sql
CREATE TABLE public.transcript_analyses (
    id                     uuid NOT NULL,
    transcript_text        text NOT NULL,        -- formatted/diarized text
    raw_transcript         text,                 -- original unprocessed text
    gemini_response        jsonb NOT NULL,       -- full LLM extraction JSON
    applied_changes        jsonb,                -- what was actually applied
    status                 text DEFAULT 'pending',
    source                 text DEFAULT 'manual',-- recall | meet | manual | mcp | file
    title                  text,
    deal_id                uuid,                 -- nullable link to the record
    workspace_id           uuid NOT NULL,
    created_by             uuid NOT NULL,
    created_at             timestamptz DEFAULT now(),
    audio_url              text,
    audio_duration_secs    integer,
    audio_s3_key           text,
    video_s3_key           text,
    media_archive_status   text,                 -- processing|done|failed|skipped
    media_archived_at      timestamptz,
    recall_media_deleted_at timestamptz,
    translated_transcript  text,                 -- multilingual support
    metadata               jsonb
);

-- Google Meet ingestion queue
CREATE TABLE public.meet_transcript_syncs (
    id                   uuid DEFAULT gen_random_uuid() NOT NULL,
    workspace_id         uuid NOT NULL,
    user_id              uuid NOT NULL,
    conference_record_id text NOT NULL,
    meeting_title        text,
    meeting_start        timestamptz,
    meeting_end          timestamptz,
    transcript_text      text,
    participant_emails   jsonb,
    matched_deals        jsonb,
    analysis_id          uuid,                   -- → transcript_analyses
    deal_id              uuid,
    status               text DEFAULT 'pending', -- pending|processing|completed|failed|skipped
    error_message        text,
    created_at           timestamptz DEFAULT now()
);

-- Recorder-bot tracking
CREATE TABLE recall_bots (
    id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    recall_bot_id    text UNIQUE NOT NULL,
    workspace_id     uuid NOT NULL,
    user_id          uuid,
    deal_id          uuid,
    calendar_event_id text,
    meeting_url      text NOT NULL,
    bot_name         text NOT NULL DEFAULT 'Notetaker',
    status           text NOT NULL DEFAULT 'dispatching',
    status_sub_code  text,
    analysis_id      uuid REFERENCES transcript_analyses(id) ON DELETE SET NULL,
    metadata         jsonb NOT NULL DEFAULT '{}'::jsonb,
    created_at       timestamptz NOT NULL DEFAULT now(),
    updated_at       timestamptz NOT NULL DEFAULT now()
);
```

Notes and other activity live in a generic `engagements` table (`type` discriminator) joined to deals/contacts through an `associations` table — so a transcript-derived note is the same shape as a hand-written one.

---

## 6. RAG / Embeddings Layer

### 6.1 Vector store

```sql
CREATE TABLE public.embeddings (
    id           uuid DEFAULT gen_random_uuid() NOT NULL,
    source_type  text NOT NULL,        -- see §6.5
    source_id    uuid NOT NULL,        -- PK in the source table
    chunk_index  integer DEFAULT 0 NOT NULL,
    chunk_text   text NOT NULL,
    embedding    vector(768) NOT NULL, -- pgvector
    workspace_id uuid NOT NULL,        -- tenant isolation (indexed)
    deal_id      uuid,                 -- DENORMALIZED for fast deal-scoped search
    created_at   timestamptz DEFAULT now(),
    updated_at   timestamptz DEFAULT now()
);

-- Durable work queue (embeddings generated async, not on the request path)
CREATE TABLE embedding_queue (
    id          BIGSERIAL PRIMARY KEY,
    source_type TEXT NOT NULL,
    source_id   TEXT NOT NULL,
    raw_text    TEXT NOT NULL,
    workspace_id UUID NOT NULL,
    user_id     UUID,                  -- usage attribution (who triggered it)
    queued_at   TIMESTAMPTZ DEFAULT now()
);
```

### 6.2 Model & chunking

- **Embedding model:** `gemini-embedding-001`, output dimension **768** (must match the column).
- **Chunking:** `chunk_text(text, max_tokens=512, overlap=100)` — ~512-token windows with 100-token overlap so context isn't severed mid-thought.
- **Task types:** documents embedded with `RETRIEVAL_DOCUMENT`, queries with `RETRIEVAL_QUERY` (asymmetric retrieval).

### 6.3 Async pipeline

Writers call `queue_embeddings(source_type, source_id, raw_text, workspace_id, user_id, deal_id)` — **non-blocking**, it just enqueues (deleting any pending row for the same source first, for dedupe). A scheduler `flush_embedding_queue()` runs every **15 min**, takes up to 30 sources, chunks + embeds each, and upserts vectors into `embeddings` with `deal_id` preserved. This keeps the user-facing request fast and makes embedding generation crash-safe / retryable.

### 6.4 Usage attribution

Embedding token usage is metered into a shared `ai_token_usage` table (`model, caller, input_tokens, output_tokens, cached_tokens, workspace_id, user_id`). The `user_id` was added to `embedding_queue` specifically so background embeds can be attributed to the person who triggered them (previously the vast majority of embedding spend was unattributed).

### 6.5 What gets embedded

| `source_type` | Source table | Embedded text |
|---------------|--------------|----------------|
| `transcript` | `transcript_analyses` | AI summary (decisions, action items, discussion points) |
| `transcript_raw` | `transcript_analyses` | verbatim transcript |
| `note` | `notes` / `engagements` | note body |
| `deal` | `deals` | name, stage, amount, pipeline |
| `contact` | `contacts` | name, title, company |
| `email` | `gmail_messages` | subject + body (when matched to a deal) |
| `prd` | `prd_versions` | scope/PRD content |
| `engagement` | `engagements` | activity payload |
| `chat` | `chat_messages` | prior Q&A turns |

### 6.6 Semantic search

```sql
SELECT source_type, source_id, chunk_index, chunk_text, deal_id,
       1 - (embedding <=> %(qvec)s::vector) AS similarity
FROM embeddings
WHERE workspace_id = %(ws)s::uuid
  [AND deal_id = %(deal)s::uuid]      -- optional deal scope
  [AND source_type = %(stype)s]       -- optional type filter
ORDER BY embedding <=> %(qvec)s::vector   -- cosine distance, nearest first
LIMIT %(k)s;
```

`search_similar(query, workspace_id, deal_id?, source_type?, limit, user_id)` embeds the query then runs the above. **Crucial visibility rule:** the query returns `deal_id`, and callers resolve final visibility from the **source table's** deal_id, *not* from `embeddings.deal_id` (which is denormalized and not always populated for every source). Over-fetch (e.g. `limit × 3`) before applying per-user visibility filtering so you still return a full page after dropping hidden rows.

---

## 7. Chat — "Ask anything"

Two surfaces share the same retrieval core: a programmatic **MCP tool** and an interactive **deal copilot chat**.

### 7.1 MCP tool: `ask_mevak`

```
ask_mevak(api_key, question, deal_name?, source_type?, limit=8)
```

Resolves the deal (if named) with visibility checks, over-fetches `search_similar()` results, resolves each chunk to a **citation label**, filters by the user's visible deals, and returns up to `limit` cited snippets. Citation label format:

```
{SOURCE_TYPE} · {DEAL_NAME | TITLE} · {DD MMM YYYY}
  e.g.  Transcript · Acme Corp · 14 Jun 2026
        Email · "Re: pricing" · 11 Jun 2026
        PRD · Acme Corp · v3
```

### 7.2 Deal copilot chat (agentic)

| Endpoint | Purpose |
|----------|---------|
| `POST /api/deals/{id}/chat` `{ question }` | One turn → `{ message_id, answer, citations, proposed_action? }` |
| `GET /api/deals/{id}/chat` | Conversation history (user-scoped, most-recent N) |
| `POST /api/deals/{id}/chat/{message_id}/apply` | Execute the proposed action (create_action / draft_email / update_stage / update_closedate), with a double-apply guard + audit log |

**Turn engine** (`chat_turn()`):

1. **Deal spine** — fetch structured facts: deal basics, contacts + MEDDIC roles, open actions, computed health signals (past close date, no actions).
2. **RAG** — `retrieve_deal_context(deal_id, question)` embeds the question and pulls the top ~8 chunks scoped to this deal, hydrating each into a citation (label + date + deep-link).
3. **Prompt** — system prompt pins today's date, forbids fabricating facts, and supplies the spine (JSON) + retrieved chunks + last ~8 conversation turns; the model may call **at most one** tool from a small tool set.
4. **Model** — Gemini with function-calling (`caller="deal_copilot"`, dynamic thinking on).
5. **Response** — either a grounded JSON answer with `used_refs` (citations filtered to those actually used), or a `proposed_action` the user confirms before anything mutates.

### 7.3 Conversation persistence

```sql
CREATE TABLE public.chat_messages (
    id               uuid NOT NULL,
    workspace_id     uuid NOT NULL,
    user_email       text NOT NULL,
    deal_id          uuid,
    message_type     text NOT NULL DEFAULT 'text',   -- text | voice | copilot
    intent           text,        -- question|update|command|meeting_prep|enrich|copilot
    user_text        text,
    ai_response      text,
    proposed_changes jsonb,       -- citations + proposed_action awaiting confirmation
    applied_changes  jsonb DEFAULT '[]',  -- what was executed on /apply
    created_at       timestamptz DEFAULT now()
);
```

Chat turns are themselves embedded (`source_type='chat'`), so the assistant can recall earlier conversations. Responses are computed fully before returning (no SSE streaming in the reference build — a clean place to add it).

### 7.4 Frontend

A `DealCopilot` component renders the conversation, suggestion chips, and **proposed-action confirmation cards** (e.g. an editable draft email, a create-action card). A `ChatMessages` component renders markdown answers with clickable citations and an applied-changes summary. React hooks wrap the three endpoints (`useDealChatHistory`, `useSendChat`, `useApplyChatAction`).

---

## 8. Backend API Summary

| Method & Path | Purpose |
|---------------|---------|
| `POST /api/recall/bots` | Dispatch a recorder bot to a meeting |
| `POST /api/webhooks/recall` | Recorder webhook (live transcript + finalize) |
| `GET /api/meet-transcripts/pending` | List Google Meet transcripts awaiting confirmation |
| `GET /api/meet-transcripts/{id}` | Get one synced transcript |
| `POST /api/meet-transcripts/{id}/ingest` | Confirm + analyze a Meet transcript |
| `POST /api/meet-transcripts/{id}/skip` | Dismiss a Meet transcript |
| `GET /api/deals/{id}/pending-transcripts` | Meet transcripts matching this deal's contacts |
| `POST /api/analyze-transcript` | Manual paste → analyze |
| `POST /api/analyze-transcript-file` | File upload (≤20 MB) → analyze |
| `POST /api/deals/{id}/chat` · `GET` · `/apply` | Deal copilot chat turn / history / apply action |
| MCP `submit_transcript` | Programmatic transcript submission |
| MCP `ask_mevak` | Cited semantic Q&A across the collection |

---

## 9. Integrations & Environment

| Service | Use | Env |
|---------|-----|-----|
| **Recall.ai** | Meeting-recorder bot, diarized transcript, media | `RECALL_API_KEY`, `RECALL_REGION_BASE_URL`, `RECALL_WEBHOOK_SECRET`, `RECALL_DEFAULT_BOT_NAME` |
| **Google Gemini** | Extraction LLM (`gemini-2.5-flash-lite`), embeddings (`gemini-embedding-001`), Files API | `GOOGLE_API_KEY` |
| **Google Meet / Calendar** | Native transcript sync, event titles, participants | Per-user Google OAuth |
| **S3 (or compatible)** | Transcript media, uploaded files, screenshots | `S3_ENDPOINT`, `S3_BUCKET` (+ `UPLOAD_DIR` local fallback) |
| **Postgres + pgvector** | Collection + vector store | — |
| **Slack** *(optional)* | DM the user the actions created from a meeting | Slack token |

---

## 10. Cost & Usage Controls

- **Thinking-budget tiers** — a `THINKING_CALLERS` allowlist enables dynamic LLM "thinking" only where reasoning pays off (transcript analysis, copilot chat); every other caller injects `thinking_budget=0`. Thinking tokens bill as expensive output, so a tiny reply with thinking on can cost 30× a normal one — gate it deliberately.
- **Per-caller metering** — every LLM and embedding call writes `ai_token_usage` rows tagged by `caller`, `workspace_id`, and `user_id`, enabling per-tenant/per-user cost dashboards and quota enforcement.
- **Async embeddings** — generated off the request path in 15-min batches; a Cloud Batch API path (≈50% cheaper) is a natural optimization for latency-tolerant work (the reference build uses Batch for email categorization, sync calls for transcripts where the user is waiting).

---

## 11. Data Collected Per Transcript

| Data Point | Source | Auto/Manual |
|------------|--------|-------------|
| Raw transcript text | Recorder / Meet / paste / file / API | Both |
| Diarized speakers + timestamps | Recorder bot | Auto |
| Meeting title, start/end, participants | Calendar / recorder / API header | Auto |
| Audio/video media | Recorder bot → S3 | Auto |
| Meeting summary | LLM | Auto |
| Action items (owner, deadline) | LLM | Auto |
| Notes / key points | LLM | Auto |
| Contacts + MEDDIC roles | LLM | Auto (safe) / Review (ambiguous) |
| Personal intel (hobbies, family, key dates) | LLM (explicit mentions only) | Auto |
| Deal field updates (stage/amount/close) | LLM | Review |
| Sentiment / risk signals | LLM | Auto |
| Embeddings (chunks → vectors) | Embedding model | Auto |
| Token usage (LLM + embedding) | Server metering | Auto |
| Workspace / user / source | Auth + ingestion path | Auto |

---

## 12. Porting Checklist

To replicate this system in another application:

**Ingestion**
- [ ] **Single analysis entry point** — funnel every source into one `run_transcript_analysis(text, workspace_id, user_id, record_id?, source)` so behavior is identical everywhere
- [ ] **Recorder-bot integration** — dispatch endpoint + webhook; verify the webhook signature; atomically claim finalization to avoid double-processing
- [ ] **Media offload** — archive recorder audio/video to your own object store; allow deleting it from the recorder to cut retention cost
- [ ] **Native-transcript poller** — background job to pull Google Meet (or equivalent) transcripts; auto-match the record by attendees + title; queue for user confirmation
- [ ] **Manual paste + file upload** — text endpoint and a multipart endpoint (convert DOCX; attach other files to the LLM via a Files API); enforce a size cap
- [ ] **Programmatic submission** — an API/MCP tool with key-based auth + per-user visibility enforcement

**Extraction**
- [ ] **Context-rich prompt** — inject the record list, team, existing contacts, and the target record's recent history so the model matches instead of duplicating
- [ ] **Holistic structured output** — one call extracts summary, notes, actions, events, contacts+roles, field updates, sentiment (define a strict JSON schema)
- [ ] **Deal-relevance guard** — instruct the model to match on business context, never on a coincidental name/email overlap
- [ ] **Auto-apply vs review tiers** — auto-apply low-risk items; route record-mutating changes + ambiguous entities to a review queue
- [ ] **Corroboration gate** — for automatic pipelines, only auto-apply to a record corroborated by attendees/title; demote content-only matches to review
- [ ] **Junk-entity filtering** — drop empty/placeholder names; never create internal teammates as external contacts; gate single-token/low-confidence entities to review
- [ ] **Idempotency + savepoints** — skip duplicate open actions; wrap auto-apply in a savepoint so a partial failure rolls back cleanly

**Collection & RAG**
- [ ] **Store verbatim + analysis** — keep raw text and the full LLM JSON together; track source + media keys
- [ ] **pgvector store** — `embeddings(source_type, source_id, chunk_index, chunk_text, embedding vector(N), workspace_id, deal_id)`; match dimension to the model
- [ ] **Chunk with overlap** — ~512-token windows, ~100-token overlap; embed docs and queries with the right asymmetric task types
- [ ] **Async embedding queue** — enqueue on write (dedupe by source), flush on a schedule; carry `user_id` for attribution
- [ ] **Multi-source embedding** — embed transcripts, notes, records, contacts, emails, docs, and chat turns into one index
- [ ] **Visibility-correct search** — resolve final visibility from the source table's owner/record, not the denormalized vector row; over-fetch before filtering

**Chat**
- [ ] **Shared retrieval core** — one semantic-search function behind both a programmatic tool and the interactive chat
- [ ] **Cited answers** — return source-typed citation labels (`TYPE · NAME · DATE`); filter to citations actually used
- [ ] **Structured spine + RAG** — combine a deterministic record "spine" (facts) with retrieved chunks (context) in the prompt
- [ ] **Agentic actions with confirmation** — let the assistant propose at most one action; persist it as a proposed change; mutate only on explicit user apply (with a double-apply guard + audit log)
- [ ] **Persist + embed conversations** — store chat turns and embed them so the assistant can recall past discussion
- [ ] **Cost controls** — gate LLM "thinking" to callers that need it; meter every LLM/embedding call per workspace + user

---

<!-- Part of the ShipFactory feature spec library — https://github.com/vishalquantana/shipfactory -->

## About Us

We are [Quantana](https://quantana.com.au), an AI-first design and development agency working with Fortune 500s to build bespoke AI solutions and provide the audit and training needed to ensure success. [Click here to learn more](https://quantana.com.au).

## License

MIT
