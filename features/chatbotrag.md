# AI Chatbot with RAG — Product Requirements Document

> A generalised, portable pattern for building an intent-routed AI chatbot backed by a Retrieval-Augmented Generation (RAG) pipeline using PostgreSQL pgvector. This PRD is prescriptive enough that any team can implement it from scratch without guessing.

---

## 1. What This Is

A floating-panel AI assistant embedded in your app that classifies every user message into an **intent** (question, data update, command, or meeting prep), assembles relevant context via RAG similarity search, and routes to a specialised handler. The user can type, speak, or attach files. When the AI proposes data changes, the user reviews and approves them before anything is written.

Think of it as a **context-aware command center** — the chatbot IS the navigation + search + data entry layer.

---

## 2. Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Browser (Client)                              │
│                                                                      │
│  ┌─────────────────────┐  REST API   ┌─────────────────────────────┐ │
│  │  Floating Chat Panel │◄──────────►│  Your App Server             │ │
│  │  (FAB + Input + Msgs)│            │  POST /api/chat/message      │ │
│  └──────────┬──────────┘            │  POST /api/chat/apply        │ │
│             │                        │  POST /api/chat/voice        │ │
│             │ useChat hook           │  POST /api/chat/upload-file  │ │
│             │ (state + mutations)    │  POST /api/chat/dedup-check  │ │
│             │                        │  GET  /api/chat/history      │ │
│             │                        └──────────┬──────────────────┘ │
└─────────────┘                                   │                    │
                                                  │                    │
┌─────────────────────────────────────────────────▼────────────────────┐
│  Backend Services                                                     │
│                                                                      │
│  ┌───────────────┐  ┌───────────────────┐  ┌──────────────────────┐ │
│  │ Intent Router  │  │ RAG Embeddings    │  │ Background Scheduler │ │
│  │ classify →     │  │ chunk → embed →   │  │ flush queue every    │ │
│  │ question /     │  │ upsert / search   │  │ 15 min, batch of 30  │ │
│  │ update /       │  │                   │  │                      │ │
│  │ command /      │  │ pgvector cosine   │  │ APScheduler or       │ │
│  │ meeting_prep   │  │ similarity search │  │ Celery Beat          │ │
│  └───────┬───────┘  └────────┬──────────┘  └──────────┬───────────┘ │
│          │                   │                         │             │
│  ┌───────▼───────────────────▼─────────────────────────▼───────────┐ │
│  │  PostgreSQL + pgvector                                           │ │
│  │  chat_messages │ embeddings (vector 768) │ embedding_queue       │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │  LLM Provider (Gemini / OpenAI / Anthropic)                      │ │
│  │  - Chat completions (intent classify, Q&A, transcript analysis)  │ │
│  │  - Embeddings (768-dim vectors)                                  │ │
│  │  - Speech-to-text (voice messages)                               │ │
│  │  - File processing (PDF/DOCX analysis)                           │ │
│  │  - Function calling (global commands)                            │ │
│  └──────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

**Four moving parts:**
1. **RAG Pipeline** — chunks your app's data, embeds it via an embedding model, stores vectors in pgvector, retrieves relevant context at query time.
2. **Intent Router** — classifies each message and routes to the correct handler (question, update, command, meeting prep).
3. **Floating Chat UI** — text + voice + file input, markdown output, proposed-changes review panel.
4. **Background Scheduler** — drains the embedding queue asynchronously so user-facing saves never block on the embedding API.

---

## 3. Database Schema

### 3.1 Chat Messages

```sql
CREATE TABLE chat_messages (
    id              UUID NOT NULL PRIMARY KEY,
    tenant_id       UUID NOT NULL,                        -- workspace/org isolation
    user_email      TEXT NOT NULL,
    record_id       UUID,                                 -- nullable, scoped to a specific record (deal, project, etc.)
    message_type    TEXT NOT NULL DEFAULT 'text',          -- 'text' | 'voice'
    intent          TEXT,                                  -- 'question' | 'update' | 'command' | 'meeting_prep'
    user_text       TEXT,
    ai_response     TEXT,
    applied_changes JSONB DEFAULT '[]',
    proposed_changes JSONB,
    created_at      TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_chat_messages_tenant ON chat_messages(tenant_id, user_email);
CREATE INDEX idx_chat_messages_record ON chat_messages(record_id);
```

### 3.2 Embeddings (pgvector)

```sql
-- REQUIRED: install pgvector first
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE embeddings (
    id            UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    source_type   TEXT NOT NULL,                           -- e.g. 'note', 'transcript', 'contact', 'record'
    source_id     UUID NOT NULL,                           -- FK to the source record
    chunk_index   INTEGER NOT NULL DEFAULT 0,              -- order within the chunked document
    chunk_text    TEXT NOT NULL,                           -- raw text of this chunk
    embedding     vector(768) NOT NULL,                   -- embedding vector (768 for Gemini, 1536 for OpenAI)
    tenant_id     UUID NOT NULL,                           -- tenant isolation
    created_at    TIMESTAMPTZ DEFAULT now(),
    updated_at    TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_embeddings_source ON embeddings(source_type, source_id);
CREATE INDEX idx_embeddings_tenant ON embeddings(tenant_id);
```

**Important:** Add an HNSW index when the table exceeds ~100K rows. Without it, every search is a full table scan:

```sql
-- Only add when needed — HNSW builds are slow on large tables
CREATE INDEX idx_embeddings_hnsw ON embeddings
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
```

### 3.3 Embedding Queue

```sql
CREATE TABLE embedding_queue (
    id            BIGSERIAL PRIMARY KEY,
    source_type   TEXT NOT NULL,
    source_id     TEXT NOT NULL,
    raw_text      TEXT NOT NULL,
    tenant_id     UUID NOT NULL,
    queued_at     TIMESTAMPTZ DEFAULT now()
);
```

**Why a queue?** Embedding API calls take 200-400ms each. Queuing decouples the save from the embed so user-facing writes return instantly. A background job drains the queue every N minutes.

---

## 4. RAG Pipeline

### 4.1 Chunking

Split long text into overlapping windows. The overlap ensures that context at chunk boundaries isn't lost.

```python
def chunk_text(text: str, max_tokens: int = 512, overlap: int = 100) -> list[str]:
    """Split text into overlapping chunks. Approximates 1 token ~ 1 word."""
    if not text:
        return []
    words = text.split()
    if len(words) <= max_tokens:
        return [text]
    chunks = []
    start = 0
    while start < len(words):
        end = start + max_tokens
        chunk = " ".join(words[start:end])
        chunks.append(chunk)
        start += max_tokens - overlap
    return chunks
```

| Setting | Value | Why |
|---|---|---|
| `max_tokens` | `512` | Fits within most embedding model context windows. Larger chunks capture more context but dilute specificity. |
| `overlap` | `100` | ~20% overlap. Prevents losing sentences that straddle chunk boundaries. |
| Tokenizer | Whitespace split | Good enough for English. Replace with `tiktoken` for OpenAI or a proper tokenizer for multilingual. |

### 4.2 Embedding

Two task types — one for indexing, one for querying:

```python
EMBEDDING_MODEL = "gemini-embedding-001"  # 768-dim
# Alternative: "text-embedding-3-small"   # OpenAI, 1536-dim

def embed_text(text: str, task_type: str = "RETRIEVAL_DOCUMENT") -> list[float]:
    """Generate embedding vector for a text chunk."""
    # --- Gemini ---
    result = client.models.embed_content(
        model=EMBEDDING_MODEL,
        contents=text,
        config={"task_type": task_type, "output_dimensionality": 768},
    )
    return result.embeddings[0].values

    # --- OpenAI alternative ---
    # result = openai.embeddings.create(model="text-embedding-3-small", input=text)
    # return result.data[0].embedding

def embed_query(query: str) -> list[float]:
    """Embed a search query (uses RETRIEVAL_QUERY task type for asymmetric search)."""
    return embed_text(query, task_type="RETRIEVAL_QUERY")
```

**Critical:** Use `RETRIEVAL_DOCUMENT` for content being indexed and `RETRIEVAL_QUERY` for the search query. This asymmetric pairing improves search quality by ~15% vs using the same task type for both. OpenAI doesn't have this distinction — use the same model for both.

### 4.3 Content Extraction

You need one extractor function per source type. This converts a DB row into embeddable text:

```python
def extract_text_for_embedding(source_type: str, row: dict) -> str | None:
    """Convert a DB row into embeddable text based on source type.

    Adapt this to YOUR data model. Add/remove source types as needed.
    """
    if source_type == "note":
        return row.get("body")

    elif source_type == "transcript":
        # Embed the AI-generated summary, not the raw transcript
        analysis = row.get("ai_analysis")
        if isinstance(analysis, dict):
            parts = []
            for key in ["summary", "key_decisions", "action_items"]:
                val = analysis.get(key)
                if val:
                    parts.append(f"{key}: {json.dumps(val) if isinstance(val, (list, dict)) else val}")
            return "\n".join(parts) if parts else json.dumps(analysis)
        return str(analysis) if analysis else None

    elif source_type == "chat":
        user = row.get("user_text", "")
        ai = row.get("ai_response", "")
        return f"User: {user}\nAI: {ai}" if user or ai else None

    elif source_type == "contact":
        parts = [row.get("name", ""), row.get("email", ""), row.get("title", ""), row.get("company", "")]
        return " | ".join(p for p in parts if p) or None

    elif source_type == "record":
        # Adapt to your primary record type (deal, project, ticket, etc.)
        parts = [
            row.get("name", ""),
            f"status: {row.get('status', '')}",
            f"value: {row.get('value', '')}",
        ]
        return " | ".join(p for p in parts if p)

    return None
```

### 4.4 Queue + Upsert

**Use the async queue in all request handlers.** Only use sync upsert for scripts/backfill.

```python
def queue_embeddings(cur, source_type: str, source_id: str, text: str, tenant_id: str) -> int:
    """Queue text for async embedding. Non-blocking — returns immediately."""
    if not text or not text.strip() or not tenant_id:
        return 0
    # Replace any pending entry for this source (prevents double-embed)
    cur.execute(
        "DELETE FROM embedding_queue WHERE source_type = %s AND source_id = %s AND tenant_id = %s::uuid",
        (source_type, str(source_id), str(tenant_id)),
    )
    cur.execute(
        "INSERT INTO embedding_queue (source_type, source_id, raw_text, tenant_id) VALUES (%s, %s, %s, %s::uuid)",
        (source_type, str(source_id), text, str(tenant_id)),
    )
    return 1


def upsert_embeddings(cur, source_type: str, source_id: str, text: str, tenant_id: str) -> int:
    """Synchronous: chunk, embed, upsert. Use for scripts only."""
    if not text or not text.strip() or not tenant_id:
        return 0
    # Delete old chunks for this source
    cur.execute(
        "DELETE FROM embeddings WHERE source_type = %s AND source_id = %s AND tenant_id = %s::uuid",
        (source_type, str(source_id), str(tenant_id)),
    )
    chunks = chunk_text(text)
    for i, chunk in enumerate(chunks):
        vector = embed_text(chunk)
        cur.execute(
            """INSERT INTO embeddings (source_type, source_id, chunk_index, chunk_text, embedding, tenant_id)
               VALUES (%s, %s, %s, %s, %s::vector, %s)""",
            (source_type, str(source_id), i, chunk, str(vector), str(tenant_id)),
        )
    return len(chunks)
```

### 4.5 Similarity Search

```python
def search_similar(cur, query: str, tenant_id: str = None, limit: int = 10) -> list[dict]:
    """Embed query and return top-K similar chunks via cosine similarity."""
    query_vector = embed_query(query)

    sql = """
        SELECT source_type, source_id, chunk_index, chunk_text,
               1 - (embedding <=> %s::vector) AS similarity
        FROM embeddings
    """
    params = [str(query_vector)]

    if tenant_id:
        sql += " WHERE tenant_id = %s::uuid"
        params.append(str(tenant_id))

    sql += " ORDER BY embedding <=> %s::vector LIMIT %s"
    params.extend([str(query_vector), limit])

    cur.execute(sql, params)
    return [dict(row) for row in cur.fetchall()]
```

**`<=>` is pgvector's cosine distance operator.** `similarity = 1 - distance`. A similarity of 1.0 is identical; 0.0 is orthogonal.

### 4.6 Background Scheduler

Runs every 15 minutes. Grabs a batch from the queue, chunks, embeds, inserts.

```python
BATCH_SIZE = 30  # sources per run (each may produce multiple chunks)

def flush_embedding_queue():
    """Called by APScheduler / Celery Beat. Drains pending embedding jobs."""
    with db() as conn:
        cur = conn.cursor()

        # Atomic grab — FOR UPDATE SKIP LOCKED prevents concurrent double-processing
        cur.execute("""
            DELETE FROM embedding_queue
            WHERE id IN (
                SELECT id FROM embedding_queue
                ORDER BY queued_at
                LIMIT %s
                FOR UPDATE SKIP LOCKED
            )
            RETURNING id, source_type, source_id, raw_text, tenant_id
        """, (BATCH_SIZE,))
        rows = cur.fetchall()

        if not rows:
            return

        for row in rows:
            # Delete stale vectors, then re-embed
            cur.execute(
                "DELETE FROM embeddings WHERE source_type = %s AND source_id = %s AND tenant_id = %s::uuid",
                (row["source_type"], row["source_id"], str(row["tenant_id"])),
            )
            chunks = chunk_text(row["raw_text"])
            for i, chunk in enumerate(chunks):
                try:
                    vector = embed_text(chunk)
                    cur.execute(
                        """INSERT INTO embeddings (source_type, source_id, chunk_index, chunk_text, embedding, tenant_id)
                           VALUES (%s, %s, %s, %s, %s::vector, %s)""",
                        (row["source_type"], row["source_id"], i, chunk, str(vector), str(row["tenant_id"])),
                    )
                except Exception as e:
                    logger.error("embed failed for %s/%s chunk %d: %s", row["source_type"], row["source_id"], i, e)

        conn.commit()
```

**Register with APScheduler:**
```python
from apscheduler.schedulers.background import BackgroundScheduler

scheduler = BackgroundScheduler()
scheduler.add_job(flush_embedding_queue, 'interval', minutes=15, id='embed_flush')
scheduler.start()
```

### 4.7 Backfill Script

Run once after initial deployment to embed all existing data:

```python
#!/usr/bin/env python3
"""One-time backfill: embed all existing content."""

# Define your source types and their tables
SOURCES = [
    ("note",       "notes"),
    ("transcript", "transcript_analyses"),
    ("chat",       "chat_messages"),
    ("contact",    "contacts"),
    ("record",     "records"),       # adapt to your primary entity
]

def backfill():
    conn = get_connection()
    cur = conn.cursor()
    total = 0
    for source_type, table in SOURCES:
        cur.execute(f"SELECT * FROM {table}")
        rows = cur.fetchall()
        for row in rows:
            text = extract_text_for_embedding(source_type, row)
            if text:
                n = upsert_embeddings(cur, source_type, str(row["id"]), text, row["tenant_id"])
                total += n
        conn.commit()
    print(f"Total chunks embedded: {total}")
```

---

## 5. Intent Classification

Every message is classified before routing. Use fast-path heuristics to skip the LLM call when intent is obvious.

### 5.1 Fast-Path Rules (No LLM Call)

```python
def classify_intent_fast(text: str, has_file: bool, has_event_id: bool) -> str | None:
    """Return intent if deterministic, None if LLM needed."""
    stripped = text.strip().lower()

    if has_event_id:
        return "meeting_prep"
    if has_file:
        return "update"
    if len(text) > 2000:
        return "update"
    if len(text) > 500 and not any(w in stripped for w in ["prep", "meeting prep", "brief me"]):
        return "update"
    if len(stripped) < 20 and not any(w in stripped for w in [
        "deal", "meeting", "call", "update", "move", "change", "delete",
        "mark", "complete", "reassign", "prep", "prepare", "brief"
    ]):
        return "question"

    return None  # needs LLM classification
```

### 5.2 LLM Classification (Fallback)

```python
CLASSIFY_PROMPT = """Classify this message as "update", "question", "command", or "meeting_prep".

An "update" is when someone shares substantive information, reports on a meeting, or gives a status update.
A "question" is when someone asks for information, greets you, or makes small talk.
A "command" is when someone instructs you to modify, change, move, delete, complete, or reassign existing data.
A "meeting_prep" is when someone asks to prepare for a meeting or wants context before a call.

If ambiguous, default to "question".

Message: "{text}"

Respond with ONLY one of: "update", "question", "command", "meeting_prep"."""

async def classify_intent(text: str) -> str:
    fast = classify_intent_fast(text, has_file=False, has_event_id=False)
    if fast:
        return fast

    result = await llm_generate(CLASSIFY_PROMPT.format(text=text), timeout=30)
    intent = result.strip().lower()
    return intent if intent in ("update", "question", "command", "meeting_prep") else "question"
```

### 5.3 Intent → Handler Routing

| Intent | Handler | User Confirmation? | Description |
|---|---|---|---|
| `question` | `answer_question()` | No | RAG search + structured data → LLM answer |
| `update` | `analyze_update()` | **Yes** — proposed changes shown first | Extract structured data → user selects → apply |
| `command` | `execute_command()` | No — immediate | Modify existing records (move deadlines, complete actions) |
| `meeting_prep` | `meeting_prep_brief()` | No | Calendar lookup → CRM context → brief |

---

## 6. Intent Handlers

### 6.1 Question Handler

Assembles context from multiple sources, adds RAG results, sends to LLM.

```python
async def answer_question(question: str, user: dict, record_id: str = None) -> tuple[str, bool]:
    """Answer a question using structured data + RAG context.

    Returns (answer_text, unable_to_answer).
    """
    tenant_id = user["tenant_id"]
    context_parts = []

    with db() as conn:
        cur = conn.cursor()

        # 1. Record-specific context (if scoped to a record)
        if record_id:
            # Fetch the record, its notes, actions, contacts, etc.
            # ... adapt to your schema ...
            pass

        # 2. Global context (top records, open actions, upcoming events)
        # ... adapt to your schema ...

        # 3. RAG: retrieve semantically relevant chunks
        try:
            rag_results = search_similar(cur, question, tenant_id=tenant_id, limit=10)
            if rag_results:
                rag_chunks = [f"[{r['source_type'].upper()}] {r['chunk_text']}" for r in rag_results]
                context_parts.append(
                    "Semantically relevant context:\n" + "\n---\n".join(rag_chunks)
                )
        except Exception as e:
            logger.warning(f"RAG search failed, continuing with SQL context: {e}")

    context = "\n\n".join(context_parts) if context_parts else "No context available."

    prompt = f"""You are an AI assistant for [Your App Name].
Answer the user's question using the context provided.
Be concise and direct. Use specific numbers and names from the data.
If the data doesn't contain enough info, say "I don't have enough data" and explain what's missing.
Today's date is {date.today().isoformat()}.

Context:
{context}

Question: {question}

Answer:"""

    answer = await llm_generate(prompt, timeout=30)

    # Detect inability to answer
    unable = any(phrase in answer.lower()[:120] for phrase in [
        "i don't have enough data",
        "i don't have enough information",
        "i cannot answer",
    ])

    return answer, unable
```

**After answering, embed the chat exchange for future RAG retrieval:**

```python
# Embed the Q&A pair so future questions can find it
chat_text = f"User: {question}\nAI: {answer}"
queue_embeddings(cur, "chat", str(message_id), chat_text, str(tenant_id))
```

### 6.2 Update Handler (Transcript / Data Extraction)

The most complex handler. Extracts structured data from unstructured text, then presents proposed changes for user approval.

**Step 1: Build the extraction prompt**

```python
EXTRACTION_PROMPT = """You are a data analyst for [Your App Name].
Analyze the text and extract structured data.

TODAY'S DATE: {today}

EXISTING RECORDS:
{records_json}

TEAM MEMBERS:
{members_json}

EXISTING CONTACTS:
{contacts_json}

INSTRUCTIONS:
1. Match mentions to existing records by name. Use record IDs from the list.
2. If a new entity is discussed, set record_id to null and is_new to true.
3. For each record, extract:
   - summary: 1-2 sentence summary
   - notes: key discussion points (array of strings)
   - actions: action items with title, owner_name, deadline (YYYY-MM-DD)
   - contacts: people mentioned with name, title, company, role
   - field_updates: any status/value changes detected
4. Only include fields with actual content from the text.

Return ONLY valid JSON matching this schema:
{{"records": [
  {{
    "record_id": "uuid or null",
    "record_name": "string",
    "is_new": false,
    "proposed_changes": {{
      "summary": "string",
      "notes": ["string"],
      "actions": [{{"title": "string", "owner_name": "string", "deadline": "YYYY-MM-DD"}}],
      "contacts": [{{"name": "string", "title": "string", "company": "string", "role": "string"}}],
      "field_updates": {{"status": "string", "value": 0}}
    }}
  }}
]}}

TEXT:
{text}"""
```

**Step 2: Post-process into proposed changes with dedup**

```python
def build_proposed_changes(analysis: dict, existing_data: dict) -> list[dict]:
    """Convert LLM analysis into a list of proposed changes with dedup.

    Each change has: type, record_name, record_id, summary, data.
    """
    proposed = []
    existing_action_keys = {(a["title"].lower(), a["record_id"]) for a in existing_data["actions"]}
    existing_contact_keys = {c["name"].lower() for c in existing_data["contacts"]}

    for record_data in analysis.get("records", []):
        record_name = record_data.get("record_name", "Unknown")
        record_id = record_data.get("record_id")
        changes = record_data.get("proposed_changes", {})

        # New record
        if record_data.get("is_new"):
            proposed.append({
                "type": "new_record",
                "record": record_name,
                "record_id": None,
                "summary": f"Create new record: {record_name}",
                "data": record_data,
            })

        # Notes
        for note in changes.get("notes", []):
            proposed.append({
                "type": "note",
                "record": record_name,
                "record_id": record_id,
                "summary": note[:120],
                "data": {"body": note},
            })

        # Actions (with dedup)
        for action in changes.get("actions", []):
            key = (action["title"].lower(), record_id)
            if key in existing_action_keys:
                continue  # skip duplicate
            existing_action_keys.add(key)
            proposed.append({
                "type": "action",
                "record": record_name,
                "record_id": record_id,
                "summary": f"{action['title']} → {action.get('owner_name', 'Unassigned')}",
                "data": action,
            })

        # Contacts (with dedup)
        for contact in changes.get("contacts", []):
            if contact["name"].lower() in existing_contact_keys:
                proposed.append({
                    "type": "link_contact",
                    "record": record_name,
                    "record_id": record_id,
                    "summary": f"Link existing: {contact['name']}",
                    "data": contact,
                })
                continue
            existing_contact_keys.add(contact["name"].lower())
            proposed.append({
                "type": "contact",
                "record": record_name,
                "record_id": record_id,
                "summary": f"{contact['name']}, {contact.get('title', '')}",
                "data": contact,
            })

        # Field updates
        for field, value in changes.get("field_updates", {}).items():
            proposed.append({
                "type": "field_update",
                "record": record_name,
                "record_id": record_id,
                "summary": f"{field}: → {value}",
                "data": {"field": field, "value": value},
            })

    return proposed
```

**Step 3: Return proposed changes — do NOT apply them.** The user reviews and selects which to apply via `/api/chat/apply`.

### 6.3 Apply Changes Handler

Two-pass: create new records first (so other changes can reference them), then apply everything else with savepoints.

```python
def apply_selected_changes(changes: list[dict], user: dict) -> list[dict]:
    """Apply user-approved changes to the database. Uses savepoints for safety."""
    tenant_id = user["tenant_id"]
    applied = []

    with db() as conn:
        cur = conn.cursor()

        # Pass 1: Create new records
        new_record_ids = {}
        for change in changes:
            if change["type"] != "new_record":
                continue
            name = change["data"].get("record_name", "")
            # Dedup: check if record with same name already exists
            cur.execute("SELECT id FROM records WHERE tenant_id = %s AND LOWER(name) = LOWER(%s)", (tenant_id, name))
            existing = cur.fetchone()
            if existing:
                new_record_ids[name] = existing["id"]
                continue
            new_id = str(uuid.uuid4())
            cur.execute("INSERT INTO records (id, tenant_id, name) VALUES (%s, %s, %s)", (new_id, tenant_id, name))
            new_record_ids[name] = new_id
            applied.append(change)

        # Pass 2: Apply all other changes with savepoints
        for i, change in enumerate(changes):
            if change["type"] == "new_record":
                continue
            record_id = change.get("record_id") or new_record_ids.get(change.get("record"))
            sp = f"sp_{i}"
            try:
                cur.execute(f"SAVEPOINT {sp}")

                if change["type"] == "note":
                    # Dedup: skip if similar note exists in last 24h
                    cur.execute("""
                        SELECT id FROM notes WHERE record_id = %s AND tenant_id = %s
                        AND created_at > NOW() - INTERVAL '24 hours'
                        AND LEFT(LOWER(body), 200) = LEFT(LOWER(%s), 200) LIMIT 1
                    """, (record_id, tenant_id, change["data"]["body"]))
                    if not cur.fetchone():
                        cur.execute("INSERT INTO notes (...) VALUES (...)")
                        # Queue for RAG embedding
                        queue_embeddings(cur, "note", note_id, change["data"]["body"], tenant_id)
                        applied.append(change)

                elif change["type"] == "action":
                    # Similar dedup + insert pattern
                    pass

                elif change["type"] == "contact":
                    # Insert contact + link to record + queue for embedding
                    pass

                elif change["type"] == "field_update":
                    field = change["data"]["field"]
                    if field in ALLOWED_UPDATE_FIELDS:  # whitelist!
                        cur.execute(f"UPDATE records SET {field} = %s WHERE id = %s", (change["data"]["value"], record_id))
                        applied.append(change)

                cur.execute(f"RELEASE SAVEPOINT {sp}")
            except Exception as e:
                cur.execute(f"ROLLBACK TO SAVEPOINT {sp}")
                logger.error("Failed to apply change %s: %s", change["type"], e)

        conn.commit()
    return applied
```

### 6.4 Command Handler

For direct data mutations ("move all actions to Friday", "mark overdue as done").

```python
COMMAND_PROMPT = """You are a data assistant. Execute the user's instruction.

TODAY'S DATE: {today}
CURRENT RECORD: {record_context}
CURRENT ACTIONS: {actions_json}
TEAM MEMBERS: {team_json}

User instruction: "{instruction}"

Return JSON:
{{
  "summary": "brief description of what you did",
  "operations": [
    {{
      "type": "update_action | complete_action | update_record",
      "id": "record uuid",
      "changes": {{"field": "value"}},
      "label": "short UI description"
    }}
  ]
}}"""
```

**Commands are applied immediately** (no user confirmation step). Each operation uses a savepoint.

### 6.5 Meeting Prep Handler

Fetches upcoming calendar events, matches attendees to your CRM contacts, builds context, sends to LLM with strict anti-hallucination prompt.

```python
MEETING_PREP_PROMPT = """You are a meeting preparation assistant.
Generate a concise meeting brief using ONLY the context provided below.

CRITICAL RULES:
- Do NOT fabricate or invent any information.
- If data is missing, say "No data available" for that section.
- Every fact must come from the provided context.

Format with markdown sections:
## Meeting: [title]
**Time:** [start time]
**Attendees:** [list]

### Record Context
### Recent Interactions
### Open Actions
### Suggested Talking Points (only if grounded in context above)"""
```

**Deep mode:** If the user asks for meeting prep twice within 30 minutes, the second call includes transcript insights and stakeholder mapping.

---

## 7. Three-Layer Deduplication

Duplicates are the #1 quality issue in AI-assisted data entry. Use all three layers.

### Layer 1: SQL Dedup (Real-Time, Before Showing to User)

During `analyze_update()`, check existing data before building proposed changes:

| Data Type | Dedup Key |
|---|---|
| Actions | `LOWER(title)` + `record_id` |
| Notes | First 80 chars lowercase + `record_id` |
| Events | `LOWER(title)` + `date` + `record_id` |
| Contacts | `LOWER(name)` in tenant |

### Layer 2: LLM Dedup (Background, After Showing to User)

Fire a background request after proposed changes are displayed:

```python
DEDUP_PROMPT = """Compare each NEW item against EXISTING items for the same record.
Return JSON: [{"index": N, "is_duplicate": true/false, "matched_item": "...", "confidence": 0.0-1.0}]

Only flag as duplicate if semantic meaning is essentially the same.
Do NOT flag items that are merely related but different.

EXISTING: {existing_json}
NEW ITEMS: {new_json}"""
```

**Use a cheap/fast model** (Gemini Flash Lite, GPT-4o-mini) with an 8-second timeout. Threshold: confidence >= 0.7. Frontend auto-unchecks flagged items.

### Layer 3: Apply-Time Dedup (Safety Net)

During `apply_selected_changes()`, SQL-check again right before INSERT:
- Notes: Normalized text (collapse whitespace), first 200 chars, 24h window
- Actions: Normalized title, first 100 chars, open status only
- Events: Title + date exact match

---

## 8. Prompt Injection Defence

Check every message before sending to the LLM:

```python
INJECTION_PATTERNS = [
    "ignore previous instructions", "ignore all previous", "disregard previous",
    "forget your instructions", "forget everything above", "jailbreak",
    "dan mode", "developer mode", "reveal your prompt", "print your system prompt",
    "show me your instructions", "you are now a different", "override your rules",
    "act as an unrestricted", "bypass your restrictions",
]

def is_prompt_injection(text: str) -> bool:
    lower = text.lower()
    return any(p in lower for p in INJECTION_PATTERNS)
```

If detected, return a safe refusal without calling the LLM:
```python
if is_prompt_injection(body.text):
    return {"type": "question", "response": "I can't help with that.", "proposed_changes": []}
```

---

## 9. API Endpoints

### 9.1 `POST /api/chat/message` — Main Chat

**Rate limit:** 15/minute

```json
// Request
{
  "text": "string",
  "record_id": "uuid | null",
  "page_url": "string",
  "file_uri": "string | null",
  "file_mime_type": "string | null",
  "event_id": "string | null"
}

// Response
{
  "type": "question | update | command | meeting_prep",
  "response": "AI text response",
  "proposed_changes": [
    {
      "type": "note | action | contact | field_update | new_record | link_contact | event",
      "record": "Record Name",
      "record_id": "uuid",
      "summary": "human-readable description",
      "data": {}
    }
  ],
  "applied_changes": [],
  "message_id": "uuid"
}
```

### 9.2 `POST /api/chat/apply` — Apply Selected Changes

```json
// Request
{ "changes": [/* selected proposed_changes */], "message_id": "uuid" }

// Response
{ "applied": [/* what was actually applied */], "count": 5 }
```

### 9.3 `POST /api/chat/voice` — Voice Message

**Rate limit:** 5/minute. Multipart form: `audio` (file), `record_id`, `page_url`.

Flow: Transcribe audio via LLM STT → classify intent → route to handler.

### 9.4 `POST /api/chat/transcribe` — Transcribe Only

**Rate limit:** 10/minute. Multipart form: `audio` (file).

Returns `{ "transcript": "text" }` for user to edit before sending.

### 9.5 `POST /api/chat/upload-file` — File Upload

**Rate limit:** 10/minute. Multipart form: `file` (PDF, TXT, DOCX, MD; max 20MB).

Uploads to LLM file API (Gemini Files API, or convert to text for OpenAI). DOCX files should be converted to plain text server-side.

Returns `{ "file_uri": "...", "mime_type": "...", "display_name": "..." }`.

### 9.6 `POST /api/chat/dedup-check` — Background Dedup

```json
// Request
{ "changes": [/* proposed_changes array */] }

// Response — keys are string indices into the changes array
{ "duplicates": { "0": { "matched_item": "Follow up call", "confidence": 0.85 } } }
```

### 9.7 `GET /api/chat/history` — Conversation History

Returns up to 200 most recent messages for the authenticated user.

### 9.8 `GET /api/chat/upcoming-meetings` — Meeting Auto-Surface

Returns upcoming meetings (next 2 hours) with attendee CRM data and record matches. Used by the frontend to surface ephemeral "want me to prep you?" prompts.

---

## 10. Client Component

### 10.1 Screen Layout

```
┌───────────────────────────────────────────────────────────────┐
│ ✨ AI Assistant        [Deal: Acme Corp]    [─] [□] [×]      │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│   ┌─────────────────────────────────────────┐                 │
│   │ ✨ AI Assistant                         │                 │
│   │ Share an update or ask a question.      │                 │
│   └─────────────────────────────────────────┘                 │
│                                                               │
│   ┌─────────────────────────────────────────┐                 │
│   │ [you] Just had a call with Acme. They   │                 │
│   │ want to move forward with the $50K...   │                 │
│   └─────────────────────────────────────────┘                 │
│                                                               │
│   ┌─────────────────────────────────────────┐                 │
│   │ [AI] I found 6 changes across 1 record: │                 │
│   │                                         │                 │
│   │  Review proposed changes (4/6)          │                 │
│   │  ─────────────────────────────          │                 │
│   │  Acme Corp                              │                 │
│   │  ☑ 📋 Summary — Met to discuss...      │                 │
│   │  ☑ 📝 Note — Budget approved for...    │                 │
│   │  ☑ ✅ Action — Send SOW → Sarah (Jun 5)│                 │
│   │  ☐ ✅ Action — Schedule kickoff ⚠ DUPE │                 │
│   │  ☑ 👤 Contact — Jane Doe, VP Eng       │                 │
│   │  ☑ 📊 Stage: Proposal → Negotiation    │                 │
│   │                                         │                 │
│   │  [ ✓ Apply 4 changes ]                  │                 │
│   └─────────────────────────────────────────┘                 │
│                                                               │
├───────────────────────────────────────────────────────────────┤
│  📎 report.pdf                          [×]                   │
│  [Type or paste a transcript...]     [📎] [🎤] [➤]          │
└───────────────────────────────────────────────────────────────┘
```

### 10.2 Component State Machine

```
                  ┌──────────┐
      mount ────► │  closed  │
                  └────┬─────┘
                       │ user clicks FAB / external event
                       ▼
                  ┌──────────┐
          ┌───── │   open   │ ◄──── restore from minimized
          │      └────┬─────┘
          │           │ header [─] button
          │           ▼
          │      ┌──────────┐
          │      │minimized │ (header only, clickable)
          │      └──────────┘
          │
          │ AI returns proposed_changes
          ▼
     ┌──────────┐
     │maximized │ (wider panel for reviewing changes)
     └──────────┘

     Any state ──── [×] button ──── ► closed
```

### 10.3 Chat Hook (State Management)

```javascript
// useChat.js — React hook pattern. Adapt to Vue composable / Svelte store.
export function useChat() {
  const [messages, setMessages] = useState([])
  const [recording, setRecording] = useState(false)
  const [attachedFile, setAttachedFile] = useState(null)

  // Derive record_id from current URL (e.g. /deals/:id, /projects/:id)
  const recordId = extractRecordIdFromUrl()

  const sendMessage = async (text, fileOverride) => {
    const file = fileOverride || attachedFile
    setMessages(prev => [...prev, { role: 'user', content: text, timestamp: Date.now() }])
    setAttachedFile(null)

    // Adaptive timeout: longer for file uploads and long text
    const timeoutMs = Math.min(
      (file ? 180_000 : 60_000) + Math.floor((text?.length || 0) / 50) * 1_000,
      300_000
    )

    const { data } = await api.post('/api/chat/message', {
      text,
      record_id: recordId,
      page_url: window.location.href,
      file_uri: file?.file_uri,
      file_mime_type: file?.mime_type,
    }, { timeout: timeoutMs })

    setMessages(prev => [...prev, {
      role: 'assistant',
      type: data.type,
      content: data.response,
      proposedChanges: data.proposed_changes || [],
      appliedChanges: data.applied_changes || [],
      messageId: data.message_id,
    }])

    // Fire background dedup check for proposed changes
    if (data.proposed_changes?.length > 0) {
      api.post('/api/chat/dedup-check', { changes: data.proposed_changes })
        .then(res => {
          const dupes = res.data?.duplicates
          if (dupes && Object.keys(dupes).length > 0) {
            // Mark duplicates in-place, auto-uncheck them
            setMessages(prev => prev.map(msg => {
              if (msg.messageId !== data.message_id) return msg
              const updated = msg.proposedChanges.map((c, i) => {
                const d = dupes[String(i)]
                return d ? { ...c, _duplicate: true, _confidence: d.confidence } : c
              })
              return { ...msg, proposedChanges: updated }
            }))
          }
        })
        .catch(() => {}) // dedup is best-effort
    }
  }

  const applyChanges = async (selectedChanges, messageId) => {
    const { data } = await api.post('/api/chat/apply', { changes: selectedChanges, message_id: messageId })
    // Replace proposed with applied in the message
    setMessages(prev => prev.map(msg =>
      msg.messageId === messageId
        ? { ...msg, proposedChanges: [], appliedChanges: data.applied }
        : msg
    ))
    // Invalidate relevant query caches
    queryClient.invalidateQueries({ queryKey: ['records'] })
  }

  return { messages, sendMessage, applyChanges, recording, attachedFile, /* ... */ }
}
```

### 10.4 Proposed Changes UI

Key UX patterns:
- **Group by record** — show record name as section header
- **Checkbox per change** — all checked by default
- **DUPLICATE badge** — orange, shown when LLM dedup flags it (auto-unchecked)
- **Type icons** — 📋 Summary, 📝 Note, ✅ Action, 👤 Contact, 🔗 Link, 📅 Event, 📊 Field Update, 🆕 New Record
- **Select All / None** buttons
- **Fullscreen expand** — for long lists, expand to a modal overlay
- **Apply button** — shows count, disabled when 0 selected, loading spinner during apply

### 10.5 Progress Bar (Long Operations)

For transcript analysis (>200 chars input), show estimated progress:

```javascript
// Estimate: 8s fixed overhead + 2.3ms per character
const estimatedMs = 8000 + (charCount * 2.3)

// Ease-out curve — approaches 95% but never 100% until response arrives
const progress = Math.min(0.95, 1 - Math.exp(-2.5 * (elapsed / estimatedMs)))
```

Status text steps: "Reading transcript..." → "Identifying records..." → "Extracting actions & contacts..." → "Analyzing..." → "Almost done..."

### 10.6 Voice Input

Use MediaRecorder API with transcribe-only mode:

1. **Start** → acquire mic, start MediaRecorder (WebM/MP4), show waveform + timer
2. **Stop** → POST audio to `/api/chat/transcribe`, populate text input with result
3. **User edits** → reviews/edits transcribed text, then sends as normal text message
4. **Max duration:** 5 minutes (auto-stop)

```javascript
function getAudioMimeType() {
  if (MediaRecorder.isTypeSupported('audio/webm;codecs=opus')) return 'audio/webm;codecs=opus'
  if (MediaRecorder.isTypeSupported('audio/webm')) return 'audio/webm'
  if (MediaRecorder.isTypeSupported('audio/mp4')) return 'audio/mp4'
  return null
}
```

### 10.7 File Upload

Supported: PDF, TXT, DOCX, MD (max 20MB). Auto-ingest text files (auto-send "Analyze this transcript").

Drag-and-drop on the chat panel + file picker button. Show file chip with filename + remove button before sending.

---

## 11. Environment Variables

| Variable | Example | Required | Purpose |
|---|---|---|---|
| `GOOGLE_API_KEY` | `AIza...` | Yes (if Gemini) | Gemini LLM + embeddings + STT |
| `OPENAI_API_KEY` | `sk-...` | Yes (if OpenAI) | OpenAI LLM + embeddings |
| `DATABASE_URL` | `postgresql://...` | Yes | PostgreSQL with pgvector |

**Feature flag pattern:**
```python
CHAT_ENABLED = bool(os.environ.get("GOOGLE_API_KEY") or os.environ.get("OPENAI_API_KEY"))
```

---

## 12. Common Pitfalls & Fixes

| Symptom | Cause | Fix |
|---|---|---|
| Vector search returns nothing | `pgvector` extension not installed | Run `CREATE EXTENSION IF NOT EXISTS vector` |
| Vector search is slow (>1s) | No HNSW index + large table | Add HNSW index (Section 3.2) when >100K rows |
| Duplicate notes/actions created | Only using one dedup layer | Implement all three layers (Section 7) |
| Chat blocks for 500ms+ on save | Embedding inline in request handler | Use `queue_embeddings()`, never `upsert_embeddings()` in handlers |
| Embeddings table grows unbounded | Never deleting stale chunks | `upsert_embeddings` deletes old chunks before inserting — ensure queue flush does too |
| AI hallucinates meeting context | Meeting prep prompt allows fabrication | Use strict anti-hallucination prompt: "Do NOT fabricate. Say 'No data available' if missing." |
| Proposed changes include duplicates | SQL dedup only checks exact match | Add LLM dedup layer (Layer 2) for semantic matching |
| User applies changes, nothing happens | `SAVEPOINT` rolled back silently on error | Log every rollback, surface errors to user |
| `vector(768)` type error | Dimension mismatch with your embedding model | Gemini = 768, OpenAI `text-embedding-3-small` = 1536. Match the column to your model. |
| Backfill script is painfully slow | Calling embedding API per chunk sequentially | Batch chunks, add delay between batches to avoid rate limits |
| Intent misclassification | Short commands classified as questions | Tune fast-path rules (Section 5.1) before relying on LLM classify |
| Transcript analysis times out | Default 30s timeout, large transcripts need 3-5 min | Set timeout to 290s for update handler, show progress bar on frontend |
| DOCX file upload fails | Gemini doesn't support DOCX natively | Convert DOCX → plain text server-side before sending to LLM |
| Voice recording fails on Safari | `audio/webm` not supported | Use codec detection with fallback to `audio/mp4` (Section 10.6) |

---

## 13. Embedding Model Comparison

| Model | Dimensions | Cost (per 1M tokens) | Latency | Notes |
|---|---|---|---|---|
| `gemini-embedding-001` | 768 | ~$0.15 | ~200ms | Supports `RETRIEVAL_DOCUMENT` / `RETRIEVAL_QUERY` task types |
| `text-embedding-3-small` (OpenAI) | 1536 | ~$0.02 | ~100ms | Cheapest option. No task type distinction. |
| `text-embedding-3-large` (OpenAI) | 3072 | ~$0.13 | ~150ms | Highest quality. Larger vectors = more storage. |
| `voyage-3` (Voyage AI) | 1024 | ~$0.06 | ~120ms | Good for code + technical content. |

**If switching models:** change the `vector(768)` column dimension, re-run backfill, and drop/recreate HNSW index.

---

## 14. LLM Model Comparison

| Use Case | Recommended | Alternative | Why |
|---|---|---|---|
| Intent classification | `gemini-2.5-flash` | `gpt-4o-mini` | Speed matters more than IQ for 4-way classify |
| Question answering | `gemini-2.5-flash` | `gpt-4o`, `claude-sonnet` | Good balance of speed + quality |
| Transcript extraction | `gemini-2.5-flash` | `gpt-4o` | Needs structured JSON output — test with your data |
| Dedup check | `gemini-2.0-flash-lite` | `gpt-4o-mini` | Cheapest + fastest. 8s timeout, best-effort. |
| Meeting prep | `gemini-2.5-flash` | `claude-sonnet` | Anti-hallucination matters — test both |

---

## 15. Security Considerations

- **Tenant isolation:** Every query MUST filter by `tenant_id` / `workspace_id`. A missing WHERE clause leaks cross-tenant data into RAG results.
- **Prompt injection:** Check every message before sending to LLM (Section 8). This is a blocklist, not a guarantee — layer with output validation.
- **Field update whitelist:** Never use `f"UPDATE ... SET {field}"` with unvalidated field names. Maintain an explicit allowlist of updatable fields.
- **Savepoints for apply:** Every change in `apply_selected_changes` runs inside a savepoint. One bad change shouldn't roll back the entire batch.
- **Rate limiting:** All chat endpoints should be rate-limited per-user. Embedding costs money; LLM calls cost more.
- **File upload validation:** Check MIME type, file size, and content. Never pass user-uploaded files directly to the LLM without validation.
- **Don't trust LLM output:** The LLM generates proposed changes — field names, UUIDs, amounts may be hallucinated. Validate everything server-side before writing to DB.

---

## 16. Implementation Checklist

### Database
- [ ] Install pgvector: `CREATE EXTENSION IF NOT EXISTS vector`
- [ ] Create `chat_messages` table
- [ ] Create `embeddings` table with `vector(N)` column (N = your model's dimensions)
- [ ] Create `embedding_queue` table
- [ ] Add indexes on `tenant_id`, `source_type + source_id`

### RAG Pipeline
- [ ] Implement `chunk_text()` (512 tokens, 100 overlap)
- [ ] Implement `embed_text()` + `embed_query()` for your chosen embedding model
- [ ] Implement `extract_text_for_embedding()` for each of your source types
- [ ] Implement `queue_embeddings()` (async) and `upsert_embeddings()` (sync)
- [ ] Implement `search_similar()` with cosine distance
- [ ] Set up background scheduler (APScheduler/Celery, 15-min interval, batch of 30)
- [ ] Write backfill script for existing data
- [ ] Run backfill

### Chat Backend
- [ ] Implement intent classification (fast-path rules + LLM fallback)
- [ ] Implement question handler with RAG context assembly
- [ ] Implement update handler (extraction prompt + proposed changes + dedup)
- [ ] Implement apply handler (two-pass with savepoints)
- [ ] Implement command handler (LLM → structured operations → execute)
- [ ] Implement meeting prep handler (calendar + CRM context → brief)
- [ ] Implement prompt injection check
- [ ] Implement all 8 API endpoints with rate limiting
- [ ] Implement 3-layer deduplication

### Frontend
- [ ] Build floating chat panel (FAB) with open/close/minimize/maximize states
- [ ] Build chat input (text + voice recording + file upload + drag-and-drop)
- [ ] Build message display with markdown rendering
- [ ] Build ProposedChanges component (grouped checkboxes, dedup badges, apply button)
- [ ] Build AppliedChanges component (green checkmarks, record navigation)
- [ ] Build progress bar for long operations
- [ ] Build useChat hook (state, mutations, adaptive timeout, background dedup)
- [ ] Wire up query cache invalidation after apply

### Integration Points (Adapt to Your App)
- [ ] Tenant/workspace isolation on all queries
- [ ] Auth middleware returning `{ user_id, email, tenant_id }`
- [ ] Record stages/statuses constant for extraction prompt
- [ ] Calendar integration for meeting prep (optional)
- [ ] Error ticket creation when AI can't answer (optional)
