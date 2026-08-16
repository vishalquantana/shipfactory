# Self-Learning Support Chatbot (Human-in-the-Loop FAQ) — Product Requirements Document

> A generalised, portable pattern for an intent-routed assistant that answers from a growing FAQ and live app data, and — when it can't answer — escalates the question to a human staff channel, then **harvests the human's reply into a permanent FAQ so it never has to ask again**. Deliberately RAG-lite: no vector database, no embeddings — just SQL context injection and a keyword FAQ match, which keeps it cheap enough to run on the edge. This PRD is prescriptive enough that any team can implement it from scratch without guessing.

---

## 1. What This Is

A chat assistant embedded in your product that classifies every user message into an **intent**, then either answers it directly (from a curated FAQ or by injecting live SQL data into the prompt) or, if its confidence is too low, tells the user *"let me check on that and get back to you"* and quietly escalates the question to your staff.

The escalation lands in an existing **team chat channel** — not a bespoke admin dashboard. A staff member replies in natural language, exactly as they'd answer a colleague. A background job then:

1. Writes that answer back into the original user's chat thread ("I checked — …") and pushes them a notification, and
2. **Saves the question + answer as a new FAQ row**, so the *next* person who asks gets an instant, LLM-free, human-free answer.

The selling point is the **learning flywheel**: every question a human answers once becomes knowledge the bot owns forever. Support load falls monotonically. The staff channel *is* the training interface — no one edits prompts, no one curates a knowledge base by hand.

Think of it as a **support chatbot that files its own tickets and writes its own FAQ from the replies.**

> This is the complement to a classic RAG chatbot (see [`chatbotrag.md`](./chatbotrag.md)). Where that pattern maximises answer coverage via embeddings, this one maximises answer *correctness over time* via a human-in-the-loop feedback loop, and stays radically cheaper by skipping the vector store. The two compose: use RAG for retrieval breadth, use this loop for the long tail RAG can't cover.

---

## 2. Architecture Overview

```
┌───────────────────────────────────────────────────────────────────────────┐
│                            Client (web / mobile)                           │
│                                                                            │
│  ┌──────────────────────┐   REST    ┌──────────────────────────────────┐   │
│  │  Chat UI (FAB +      │◄────────►│  App Server / Edge Worker         │   │
│  │  bot thread)         │           │  POST /bot/chat                   │   │
│  │  — bot is a pinned   │           │  POST /bot/feedback               │   │
│  │    "contact" in the  │           │  GET  /bot/history                │   │
│  │    user's inbox      │           │  GET  /bot/admin/check-replies    │   │
│  └──────────────────────┘           └───────────────┬──────────────────┘   │
└──────────────────────────────────────────────────────┼─────────────────────┘
                                                        │
┌───────────────────────────────────────────────────────▼─────────────────────┐
│  Answer Pipeline (handleChat)                                                │
│                                                                              │
│  ┌───────────────┐   ┌────────────────┐   ┌───────────────────────────────┐ │
│  │ Intent Router  │──►│ Context Builder │──►│ Answerer                      │ │
│  │ regex guards   │   │ per-intent SQL  │   │ • FAQ keyword match (no LLM)  │ │
│  │ + cheap LLM    │   │ + static blocks │   │ • or LLM(system+context+hist) │ │
│  │ classify       │   │ (live lookups)  │   │ • confidence estimate → flag  │ │
│  └───────────────┘   └────────────────┘   └───────────────┬───────────────┘ │
│                                                            │ flagged?        │
│                                                            ▼                 │
│                                              ┌───────────────────────────┐   │
│                                              │  escalateToStaff()        │   │
│                                              │  (fire-and-forget)        │   │
│                                              └─────────────┬─────────────┘   │
└────────────────────────────────────────────────────────────┼────────────────┘
                                                             │
┌─────────────────────────────────────────┐   ┌──────────────▼───────────────┐
│  Database (any SQL — Postgres / SQLite / │   │  Staff Chat Channel          │
│  libSQL / MySQL)                          │   │  (reuse your group-chat)     │
│                                           │   │  Bot posts: 'Alice asked:    │
│  chat_messages   faqs   escalations       │   │  "…" — anyone know?'         │
│  processed_replies                        │◄──┤  Staff replies inline  ───┐  │
│  + your live tables (schedule, users, …)  │   └───────────────────────────┼──┘
└────────────────────┬──────────────────────┘                              │
                     │                                                       │
         ┌───────────▼──────────────┐   Scheduled every N min                │
         │  harvestReplies() (cron)  │◄──────────────────────────────────────┘
         │  staff reply →            │
         │   1. resolve escalation   │
         │   2. answer user + push   │
         │   3. INSERT INTO faqs     │  ← the learning step
         │   4. mark processed       │
         └───────────────────────────┘
```

**Five moving parts:**

1. **Intent Router** — cheap, regex-guarded classifier that labels each message (`faq`, `schedule`, `people`, `chitchat`, `abuse`, `injection`, `complex`, …) and picks the handler + model.
2. **Context Builder** — per-intent functions that run **direct SQL** against your live tables and format the rows as plain text injected into the prompt. This is the "retrieval" — no embeddings.
3. **Answerer + Confidence Gate** — matches FAQs by keyword (no LLM) or calls an LLM with the injected context; then estimates confidence and sets a `flagged` bit when the answer is weak.
4. **Escalation** — on `flagged`, posts the question into a human staff channel and notifies staff. Fire-and-forget so the user's request never blocks.
5. **Reply Harvester** — a scheduled job that reads staff replies, delivers them to the original user, and **promotes each into a permanent FAQ row**. This is the flywheel.

**Design stance — why no vector DB:** context injection (SQL → text → prompt) covers the "live data" questions (schedule, who's who, status) exactly, with zero embedding cost and no index to maintain. The FAQ layer covers the static long tail, and the human loop grows it. You can bolt on embeddings later (see §13) but you rarely need them to ship, and skipping them lets the whole pipeline run in a single edge request.

---

## 3. Database Schema

Written in portable SQL. Types shown for PostgreSQL; SQLite/libSQL equivalents in comments. Only four new tables — everything else reuses tables you already have (users, your group-chat, your domain data).

### 3.1 Chat Messages

Every turn, user and assistant, in one log. This is also the audit trail the harvester searches.

```sql
CREATE TABLE chat_messages (
    id            TEXT PRIMARY KEY,                 -- uuid
    user_email    TEXT NOT NULL,                    -- or user_id; the thread owner
    role          TEXT NOT NULL,                    -- 'user' | 'assistant'
    body          TEXT NOT NULL,                    -- the message text
    intent        TEXT,                             -- classifier output (assistant rows)
    model_used    TEXT,                             -- which model answered (or null for FAQ/static)
    confidence    REAL,                             -- 0.0–1.0 estimate (assistant rows)
    flagged       INTEGER NOT NULL DEFAULT 0,       -- 1 = low confidence, needs a human
    resolved      INTEGER NOT NULL DEFAULT 0,       -- 1 = a human answered this flagged row
    feedback_rating   INTEGER,                      -- optional 👍/👎 (+1 / -1)
    feedback_comment  TEXT,                         -- optional free text
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()  -- SQLite: TEXT default (datetime('now'))
);

CREATE INDEX idx_chat_messages_user    ON chat_messages(user_email);
CREATE INDEX idx_chat_messages_flagged ON chat_messages(flagged, resolved);
```

### 3.2 FAQs — the knowledge base

The single source of truth the bot answers from. Rows start as hand-seeded (`manual`) and grow automatically (`human_resolved`).

```sql
CREATE TABLE faqs (
    id          INTEGER PRIMARY KEY,                -- AUTOINCREMENT / BIGSERIAL
    question    TEXT NOT NULL,
    answer      TEXT NOT NULL,
    category    TEXT DEFAULT 'general',
    source      TEXT NOT NULL DEFAULT 'manual',     -- 'manual' | 'human_resolved' | 'csv_import'
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_faqs_source ON faqs(source);
```

> **`source` is the whole story.** `manual` rows are what you seeded. `human_resolved` rows are what the flywheel produced — every one of them represents a question a human answered exactly once. Reporting on `COUNT(*) WHERE source='human_resolved'` is your "the bot learned N things" metric.

### 3.3 Escalations — link a flagged question to its staff-channel post

This table is the robust replacement for parsing the question back out of the staff message text (see the pitfall in §12). It maps *the message the bot posted into the staff channel* → *the flagged user exchange it came from*.

```sql
CREATE TABLE escalations (
    id                   TEXT PRIMARY KEY,          -- uuid
    user_email           TEXT NOT NULL,             -- who asked
    question             TEXT NOT NULL,             -- verbatim user question
    user_message_id      TEXT NOT NULL,             -- chat_messages.id of the user turn
    assistant_message_id TEXT NOT NULL,             -- chat_messages.id of the flagged answer
    staff_message_id     TEXT NOT NULL,             -- id of the bot's post in the staff channel
    status               TEXT NOT NULL DEFAULT 'open', -- 'open' | 'resolved'
    created_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at          TIMESTAMPTZ
);

CREATE INDEX idx_escalations_staff  ON escalations(staff_message_id);
CREATE INDEX idx_escalations_status ON escalations(status);
```

### 3.4 Processed Replies — idempotency ledger

The harvester runs on a timer and re-scans recent staff replies each time. This ledger guarantees each staff reply is harvested **exactly once** (no duplicate FAQ rows, no double-notifying the user).

```sql
CREATE TABLE processed_replies (
    id           TEXT PRIMARY KEY,                  -- the staff reply message id
    processed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 3.5 Tables you reuse (do not rebuild)

| Concern | Reuse |
|---|---|
| Staff escalation channel | Your existing group-chat tables (`groups`, `group_members`, `group_messages` with a `reply_to_id` self-reference). The bot is just another member. |
| Push notifications | Your existing device-token table + push sender (Expo, FCM, APNs, web-push). |
| Live-lookup data | Your domain tables — schedule, users/directory, orders, tickets, whatever the bot answers *about*. |
| The bot's identity | A reserved user row, e.g. `bot@yourapp.com`, with an avatar. It appears as a normal contact in the user's inbox and as a normal member of the staff channel. |

---

## 4. Intent Classification

Every message is classified before it's answered. Run **regex guards first** (free, instant, and a security boundary), then a cheap LLM only if needed.

### 4.1 Regex Guards (No LLM Call)

These run before anything touches an LLM. They are both a fast path and your prompt-injection / abuse boundary.

```ts
const ABUSE_PATTERNS     = [/\b(f+u+c+k|sh[i1]t|...)\b/i, /* profanity, threats */];
const IDENTITY_PATTERNS  = [/who (made|built|created) you/i, /are you (a )?(bot|ai|human)/i,
                            /what (are|r) you/i];
const INJECTION_PATTERNS = [/ignore (all |your )?previous instructions/i,
                            /reveal your (system )?prompt/i, /you are now/i,
                            /developer mode/i, /disregard/i];

function classifyGuards(text: string): Intent | null {
  if (ABUSE_PATTERNS.some(r => r.test(text)))     return 'abuse';
  if (INJECTION_PATTERNS.some(r => r.test(text))) return 'injection';
  if (IDENTITY_PATTERNS.some(r => r.test(text)))  return 'identity';
  return null; // fall through to the LLM router
}
```

### 4.2 LLM Router (Fallback)

Use your cheapest, fastest model — this is a labelling task, not reasoning. Validate the output against a fixed enum and default safely.

```ts
const ROUTER_MODEL = 'gemini-2.5-flash-lite'; // or gpt-4o-mini, claude-haiku, etc.

const VALID_INTENTS = ['faq','schedule','venue','people','networking',
                       'ventures','complex','chitchat','identity',
                       'injection','abuse'] as const;

const INTENT_PROMPT = `Classify the user's message into exactly one label:
- faq: a general "how/when/where/what" question likely answerable from a FAQ.
- schedule: asks about times, agenda, what's on / next.
- venue: asks about location, directions, parking, logistics.
- people: asks who someone is, or to find/introduce a person.
- networking: asks for suggestions on who to meet / connect with.
- ventures: asks about products, exhibitors, sponsors (adapt to your domain).
- chitchat: greeting or small talk.
- complex: a real question that spans several of the above.
- identity: asks who/what the bot is.
If unsure, answer "complex". Reply with ONLY the label.

Message: "{text}"`;

async function classifyIntent(text: string): Promise<Intent> {
  const guard = classifyGuards(text);
  if (guard) return guard;
  const raw = (await llm(ROUTER_MODEL, INTENT_PROMPT.replace('{text}', text))).trim().toLowerCase();
  return (VALID_INTENTS as readonly string[]).includes(raw) ? raw as Intent : 'complex';
}
```

### 4.3 Intent → Handler Routing

Map each intent to (a) whether it needs an LLM at all, and (b) which model.

| Intent | Answered by | LLM? | Notes |
|---|---|---|---|
| `faq` | Keyword FAQ match | **No** | Pure DB + string match. The flywheel's target. |
| `identity` | Static string | No | "I'm Meena, your event assistant…" |
| `abuse` | Static refusal | No | Don't spend tokens on it. |
| `injection` | Static refusal | No | Security boundary. |
| `chitchat` | LLM (time only in context) | Yes (cheap) | Keep it short. |
| `schedule` / `venue` / `people` / `networking` / `ventures` | LLM + live SQL context | Yes | Context Builder does the lookups. |
| `complex` | LLM + **bundled** context | Yes | FAQ + schedule + venue all injected. |

```ts
const MAIN_MODEL = 'gemini-3-flash-preview'; // your primary answer model
const INTENT_MODEL: Record<Intent, string | null> = {
  faq: null, identity: null, abuse: null, injection: null,
  chitchat: MAIN_MODEL, schedule: MAIN_MODEL, venue: MAIN_MODEL,
  people: MAIN_MODEL, networking: MAIN_MODEL, ventures: MAIN_MODEL, complex: MAIN_MODEL,
};
```

---

## 5. Context Injection (Live Lookups, No Embeddings)

The Context Builder is where "RAG" happens without a vector store: each intent has a function that queries your live tables and returns a **plain-text block** appended to the system prompt. Always stamp the current time first — it grounds every "when/next" answer.

```ts
async function buildContext(db, intent: Intent, message: string, userEmail: string): Promise<string> {
  const now = `Current time: ${formatLocalTime()}\n\n`;
  switch (intent) {
    case 'schedule':
      return now + await buildScheduleContext(db);          // SELECT from agenda tables
    case 'venue':
      return now + buildVenueContext() + await faqContext(db); // static + FAQ ride-along
    case 'people':
      return now + await buildUserProfile(db, userEmail)
                 + await buildPeopleContext(db);            // directory SELECT, LIMIT 200
    case 'networking':
      return now + await buildUserProfile(db, userEmail)
                 + await buildPeopleContext(db)
                 + NETWORKING_HINT;                         // "suggest by value, not similarity"
    case 'complex':
      return now + await faqContext(db)
                 + await buildScheduleContext(db)
                 + buildVenueContext();                     // bundle the common sources
    case 'chitchat':
    default:
      return now;
  }
}
```

**Rules that keep injected context safe and useful:**

- **Scope every query.** Filter by tenant/workspace, exclude hidden/private/soft-deleted rows, and gate on status (e.g. only `paid`/`active` records). A missing `WHERE` leaks data straight into an LLM answer.
- **Cap the size.** `LIMIT` list queries (e.g. 200 people) and truncate long fields. Injected context competes with the model's context window and your token bill.
- **Prepend the asker's own record** for personalised intents ("who should I meet") so the model reasons relative to them.
- **Format as readable text, not raw JSON dumps** — labelled lines ("Day 1 — 09:00 Keynote: …") answer better than serialized rows.

Example live-lookup builder:

```ts
async function buildScheduleContext(db): Promise<string> {
  const rows = await db.query(`
    SELECT s.day, s.start_time, s.title, GROUP_CONCAT(sp.name) AS speakers
    FROM agenda_sessions s
    LEFT JOIN agenda_speakers sp ON sp.session_id = s.id
    WHERE s.hidden = 0
    GROUP BY s.id
    ORDER BY s.day, s.start_time`);
  if (!rows.length) return 'Schedule: not published yet.\n';
  return 'Agenda:\n' + rows.map(r =>
    `Day ${r.day} — ${r.start_time} ${r.title}${r.speakers ? ` (${r.speakers})` : ''}`
  ).join('\n') + '\n';
}
```

---

## 6. The Answerer + Confidence Gate

`handleChat` is the orchestrator. It classifies, logs the user turn, answers, estimates confidence, sets `flagged`, logs the assistant turn, and returns.

```ts
async function handleChat(env, db, userEmail: string, message: string) {
  const intent = await classifyIntent(message);
  const userMsgId = await insertMessage(db, userEmail, 'user', message, { intent });

  // 1. Static / guarded intents — no LLM, never flagged.
  if (intent === 'identity')  return finish(STATIC.identity,  1.0, false);
  if (intent === 'injection') return finish(STATIC.injection, 1.0, false);
  if (intent === 'abuse')     return finish(STATIC.abuse,     1.0, false);

  // 2. FAQ intent — keyword match against the knowledge base, no LLM.
  if (intent === 'faq') {
    const hit = await findBestFaqMatch(db, message);
    if (hit) return finish(hit.answer, 0.95, false);
    return finish(STATIC.flagged, 0.2, true);   // ← no FAQ ⇒ FLAG for a human
  }

  // 3. LLM intents — inject context + short history, then call the model.
  const model = INTENT_MODEL[intent]!;
  const context = await buildContext(db, intent, message, userEmail);
  const history = await recentHistory(db, userEmail, 20);   // chronological
  const prompt = SYSTEM_PROMPT + '\n\n---\nContext:\n' + context;

  const reply = await callLLM(env, model, prompt, message, history);
  if (reply == null) return finish(STATIC.flagged, 0.1, true); // model failed ⇒ FLAG

  const confidence = estimateConfidence(reply);
  return finish(reply, confidence, confidence < 0.5);          // low conf ⇒ FLAG

  async function finish(body: string, confidence: number, flagged: boolean) {
    const asstId = await insertMessage(db, userEmail, 'assistant', body,
      { intent, model: INTENT_MODEL[intent], confidence, flagged });
    return { body, confidence, flagged, userMsgId, asstId };
  }
}
```

### 6.1 Keyword FAQ Match (no embeddings)

Deliberately simple: lowercase, split into content words, score by overlap fraction, accept the best if it clears a threshold. It catches paraphrases that share enough content words — not true semantic equivalence, but cheap, explainable, and dependency-free.

```ts
async function findBestFaqMatch(db, query: string) {
  const faqs = await db.query(`SELECT question, answer FROM faqs`);
  const qWords = tokenize(query);                 // lowercase, words length > 2, deduped
  if (!qWords.length) return null;

  let best = null, bestRatio = 0;
  for (const f of faqs) {
    const fWords = new Set(tokenize(f.question));
    const matches = qWords.filter(w => fWords.has(w)).length;
    const ratio = matches / qWords.length;        // fraction of the query covered
    if (ratio > bestRatio) { bestRatio = ratio; best = f; }
  }
  return bestRatio >= 0.4 ? best : null;          // tune 0.4 to taste
}
```

> **Why keyword and not embeddings?** The FAQ table is small (hundreds, not millions of rows) and every row is a real question a human wrote or approved. A linear scan is microseconds, needs no index or vector column, and you can *read the match logic*. Swap in embeddings (§13) only when the table grows past a few thousand rows or paraphrase misses become a real complaint.

### 6.2 Confidence Estimation → the `flagged` bit

The whole learning loop hinges on detecting "I couldn't really answer that." Three triggers set `flagged`:

```ts
const LOW_CONFIDENCE_PHRASES = [
  "let me check", "i'm not sure", "i don't have", "i'll find out",
  "not certain", "i'll get back", "i don't know",
];

function estimateConfidence(reply: string): number {
  const t = reply.toLowerCase();
  if (LOW_CONFIDENCE_PHRASES.some(p => t.includes(p))) return 0.3; // hedging ⇒ weak
  if (reply.trim().length < 20) return 0.5;                        // too terse ⇒ suspect
  return 0.9;
}
// flagged = confidence < 0.5
```

| Trigger | Where | Confidence | Flagged |
|---|---|---|---|
| FAQ intent, no keyword match ≥ 0.4 | §6 step 2 | 0.2 | ✅ |
| LLM reply contains a hedging phrase | `estimateConfidence` | 0.3 | ✅ |
| LLM call returned null (error/timeout) | §6 step 3 | 0.1 | ✅ |
| FAQ hit / clean LLM answer | — | 0.9–0.95 | ❌ |

A flagged answer shows the user a **holding message** — never a hallucinated guess:

```
STATIC.flagged = "Great question! Let me check on that and get back to you shortly."
```

The user experience is honest ("a human will follow up"), and the flag is the signal the escalation step waits for.

---

## 7. Escalation — Route the Question to a Human

When `handleChat` returns `flagged`, the request handler fires escalation **without awaiting it** (`waitUntil` on edge runtimes, or a detached promise / job on a server) so the user's HTTP response isn't delayed by staff notifications.

```ts
// in POST /bot/chat
const result = await handleChat(env, db, email, message);
if (result.flagged) ctx.waitUntil(escalateToStaff(env, db, email, message, result));
return json({ reply: result.body });
```

```ts
async function escalateToStaff(env, db, userEmail, question, result) {
  const asker  = await findProfile(db, userEmail);            // name for the message
  const channel = await getStaffChannel(db, 'Support Team');  // your group row
  if (!channel) return;

  // Post the question into the staff channel AS the bot.
  const staffMsgId = await insertGroupMessage(db, channel.id, BOT_EMAIL,
    `${asker.name} asked: "${question}" — does anyone know the answer?`);

  // Link the escalation so the harvester can find its way back (robust; see §12).
  await db.exec(`INSERT INTO escalations
      (id, user_email, question, user_message_id, assistant_message_id, staff_message_id, status)
      VALUES (?, ?, ?, ?, ?, ?, 'open')`,
    [uuid(), userEmail, question, result.userMsgId, result.asstId, staffMsgId]);

  // Notify every staff member (except the bot) via your normal push channel.
  const members = await getChannelMembers(db, channel.id);
  await Promise.all(members
    .filter(m => m.email !== BOT_EMAIL)
    .map(m => sendPush(env, m, { title: 'Support', body: `${asker.name}: "${question}"` })));
}
```

**Why a chat channel instead of an admin dashboard:** you already have group chat, push, and read receipts. Staff answer a question the same way they answer a teammate — no new UI, no training, no context-switch. The channel doubles as an audit log of every gap in the bot's knowledge.

---

## 8. Reply Harvesting — The Learning Step

A scheduled job (cron every ~10 minutes, plus a manual `GET /bot/admin/check-replies` trigger) turns staff replies into (a) delivered answers and (b) permanent FAQs.

```ts
async function harvestReplies(env, db) {
  // 1. Find replies to the bot's escalation posts, not yet processed.
  const replies = await db.query(`
    SELECT r.id, r.body, r.reply_to_id, r.sender_email
    FROM group_messages r
    JOIN group_messages parent ON parent.id = r.reply_to_id
    WHERE parent.sender_email = ?                 -- reply is to one of the bot's posts
      AND r.sender_email <> ?                     -- and not from the bot itself
      AND r.id NOT IN (SELECT id FROM processed_replies)
    ORDER BY r.created_at
    LIMIT 20`, [BOT_EMAIL, BOT_EMAIL]);

  for (const reply of replies) {
    // 2. Look up the escalation via the linked staff message (robust; no text parsing).
    const esc = await db.get(
      `SELECT * FROM escalations WHERE staff_message_id = ? AND status = 'open'`,
      [reply.reply_to_id]);
    if (!esc) { await markProcessed(db, reply.id); continue; }

    const answer = reply.body.trim();

    // 3. Resolve the flagged exchange.
    await db.exec(`UPDATE chat_messages SET resolved = 1 WHERE id = ?`, [esc.assistant_message_id]);
    await db.exec(`UPDATE escalations SET status='resolved', resolved_at=now() WHERE id = ?`, [esc.id]);

    // 4. Deliver the answer into the USER's own bot thread + notify them.
    const first = (await findProfile(db, esc.user_email)).firstName || 'there';
    await insertMessage(db, esc.user_email, 'assistant', `${first}, I checked — ${answer}`,
      { intent: 'faq', confidence: 1.0, flagged: false });
    await sendPush(env, { email: esc.user_email },
      { title: 'Support', body: answer.slice(0, 120), deepLink: '/chat/bot' });

    // 5. ⭐ THE LEARNING STEP — promote to a permanent FAQ.
    await db.exec(
      `INSERT INTO faqs (question, answer, category, source) VALUES (?, ?, 'general', 'human_resolved')`,
      [esc.question, answer]);

    // 6. Confirm back in the staff channel + record idempotency.
    await insertGroupMessage(db, /*channel*/esc_channel_id, BOT_EMAIL,
      `Thanks! I've passed that on to ${first} and added it to my FAQ.`);
    await markProcessed(db, reply.id);
  }
}
```

**What just happened, and why it matters:**

| Step | Effect |
|---|---|
| Resolve exchange | The flagged row is closed; it won't be escalated again. |
| Deliver to user | The original asker gets their answer **in the same thread**, asynchronously, with a push. No email, no ticket portal. |
| **INSERT INTO faqs** | The question is now answerable by §6.1 forever — **`source='human_resolved'`**. |
| Staff confirmation | Staff see their reply landed and taught the bot. Positive reinforcement to keep answering. |
| Mark processed | The idempotency ledger stops the timer job re-harvesting the same reply. |

### 8.1 Closing the loop — automatic re-answer

The next time *anyone* asks a question that overlaps this one, `classifyIntent` returns `faq`, `findBestFaqMatch` finds the freshly-inserted `human_resolved` row, and the user gets the answer **with no LLM call and no human involvement** at `confidence = 0.95`.

```
   Day 1                          Day 2+
┌──────────┐  flag  ┌─────────┐   ┌──────────┐  match  ┌──────────────┐
│ User asks │──────►│ Staff   │   │ User asks │────────►│ Instant FAQ   │
│ "X?"      │       │ answers │   │ "X?" (any │  0.95   │ answer, no    │
│ (flagged) │       │ once →  │   │ paraphrase)│         │ LLM, no human │
└──────────┘        │ FAQ row │   └──────────┘          └──────────────┘
                    └─────────┘
```

That is the flywheel. Human effort per distinct question trends to **once**; everything after is free.

---

## 9. Feedback Loop (Secondary)

Even confident answers can be wrong. Let users rate an answer; a 👎 escalates it to staff exactly like a flag, so bad `human_resolved`/`manual` FAQs get corrected.

```ts
// POST /bot/feedback  { messageId, rating: 1 | -1, comment? }
async function submitFeedback(db, env, messageId, rating, comment) {
  await db.exec(`UPDATE chat_messages SET feedback_rating = ?, feedback_comment = ? WHERE id = ?`,
    [rating, comment ?? null, messageId]);
  if (rating < 0) {
    const msg = await db.get(`SELECT * FROM chat_messages WHERE id = ?`, [messageId]);
    await postToStaffChannel(db, `👎 A user marked this answer unhelpful: "${msg.body}"`);
  }
}
```

To *correct* a bad FAQ, a staff member replies to that 👎 post; the harvester can `UPDATE faqs SET answer=… WHERE question=…` instead of inserting. Keep an `updated_at` bump so you can audit churn.

---

## 10. API Endpoints

### 10.1 `POST /bot/chat` — main chat

**Rate limit:** 15/min per user. Auth: the caller's session; a body `email` may only *narrow* to the authenticated user's own address, never impersonate another (403 otherwise).

```json
// Request
{ "message": "When does the keynote start?", "email": "user@x.com" }

// Response
{ "reply": "The keynote is Day 1 at 09:00 in Hall A.", "flagged": false }
```

### 10.2 `POST /bot/feedback` — rate an answer

```json
// Request
{ "messageId": "uuid", "rating": -1, "comment": "wrong room" }
// Response
{ "ok": true }
```

### 10.3 `GET /bot/history` — conversation history

Returns the authenticated user's recent turns (e.g. last 200), chronological, for rendering the thread.

### 10.4 `GET /bot/admin/check-replies` — manual harvest trigger

Staff-authenticated. Runs `harvestReplies()` on demand (so you don't wait for the cron). Returns `{ processed: N }`.

### 10.5 Scheduled trigger — the harvester

Not an HTTP endpoint. A cron entry (`*/10 * * * *`) invokes `harvestReplies()`. On edge platforms this is the platform's `scheduled` handler; on a server it's APScheduler/Celery Beat/cron.

---

## 11. Client Component

The bot is not a separate widget bolted onto the app — it's **a pinned contact in the user's existing inbox**. This is why delivered answers "just appear": they're messages in a thread the user already has open.

### 11.1 Layout

```
┌───────────────────────────────────────────────┐
│  ‹ Back      🤖 Meena  ·  Support assistant    │
├───────────────────────────────────────────────┤
│                                                │
│   ┌──────────────────────────────────────┐     │
│   │ [you] When does the keynote start?   │     │
│   └──────────────────────────────────────┘     │
│                                                │
│  ┌──────────────────────────────────────┐      │
│  │ [🤖] Great question! Let me check on  │      │
│  │ that and get back to you shortly.     │      │  ← flagged holding msg
│  └──────────────────────────────────────┘      │
│                                                │
│         · · · (minutes later, via push) · · ·  │
│                                                │
│  ┌──────────────────────────────────────┐      │
│  │ [🤖] Alex, I checked — the keynote is │      │  ← harvested answer,
│  │ Day 1 at 09:00 in Hall A.        👍 👎 │      │    arrives async
│  └──────────────────────────────────────┘      │
│                                                │
├───────────────────────────────────────────────┤
│  [Ask me anything…]                      [➤]   │
└───────────────────────────────────────────────┘
```

### 11.2 Client responsibilities

- **Send** `POST /bot/chat`, optimistically append the user bubble, then append the returned `reply`.
- **Render markdown** in bot replies (answers often contain links/lists). Parse any deep links the bot emits (e.g. `/chat/<email>`) into in-app navigation.
- **Receive async answers** through your normal message-sync path (poll `/bot/history`, socket, or push-triggered refetch) — the harvested answer is just a new `chat_messages` row.
- **Feedback affordance** — 👍/👎 under bot answers → `POST /bot/feedback`.
- **Rollout gate** — the bot is high-touch; gate visibility behind a tester allowlist / feature flag while you tune thresholds and seed FAQs.

### 11.3 Client hook (sketch)

```ts
function useBotChat(userEmail: string) {
  const [messages, setMessages] = useState<Msg[]>([]);

  const send = async (text: string) => {
    setMessages(m => [...m, { role: 'user', body: text }]);
    const { reply } = await api.post('/bot/chat', { message: text, email: userEmail });
    setMessages(m => [...m, { role: 'assistant', body: reply }]);
    // Async answers (harvested) arrive later via history sync / push refetch.
  };

  const rate = (messageId: string, rating: 1 | -1) =>
    api.post('/bot/feedback', { messageId, rating });

  return { messages, send, rate };
}
```

---

## 12. Common Pitfalls & Fixes

| Symptom | Cause | Fix |
|---|---|---|
| Harvester attaches a staff reply to the wrong question | Recovering the question by **regex on the staff message prose** (`/asked: "(.+?)" —/`) — collides when two people ask similar things | Use the `escalations` table (§3.3): link `staff_message_id → assistant_message_id`. Never parse the question back out of display text. |
| Same staff reply creates duplicate FAQs / double-notifies | No idempotency; timer re-scans the same reply each run | Insert into `processed_replies` and exclude processed ids in the harvest query (§8). |
| Bot re-escalates a question it already answered | Flagged row never marked resolved | `UPDATE chat_messages SET resolved=1` **and** `escalations.status='resolved'` on harvest. |
| Good answers get flagged | `estimateConfidence` too aggressive (short valid answers, or a hedging phrase in a correct answer) | Tune `LOW_CONFIDENCE_PHRASES` and the length threshold; log flag reasons and review the false-positive rate. |
| Bad answers *don't* get flagged | LLM confidently hallucinates without hedging phrases | Add a self-check ("if not grounded in context, reply exactly `NEEDS_HUMAN`") and treat that sentinel as a flag; layer the 👎 feedback loop (§9). |
| FAQ never matches obvious paraphrases | Keyword overlap misses synonyms ("start" vs "begin") | Lower the 0.4 threshold, add a synonym map, or graduate to embeddings (§13) once the table is large. |
| FAQ over-matches unrelated questions | Threshold too low / query dominated by common words | Raise threshold; strip stopwords in `tokenize`; require a minimum absolute match count, not just a ratio. |
| Injected context leaks private data into answers | Missing tenant / visibility / status filter in a context builder | Every context SQL filters by tenant + excludes hidden/soft-deleted + gates on status. Review each builder. |
| User's request hangs for seconds on a flagged message | Escalation (DB writes + N push sends) awaited inline | Fire escalation with `waitUntil` / detached job — never block the chat response on it. |
| Chat answer silently fails, user sees an error | LLM call throws and there's no fallback | Return the holding message + flag on any LLM null/throw (§6 step 3). The failure *becomes* an escalation, not a 500. |
| Bot answers abuse / injection attempts earnestly | Guards run after the LLM, or not at all | Run regex guards **before** any LLM call; return static refusals (§4.1). |
| Staff stop answering the channel | No feedback that their replies matter | Post the "added it to my FAQ" confirmation (§8 step 6); optionally surface a weekly "you taught the bot N answers" stat. |
| Model swap breaks classification | Router prompt tuned to one model's quirks | Keep the router prompt strict ("reply with ONLY the label") and validate against the enum with a safe default. |

---

## 13. Extension: Add Semantic Matching Later

The keyword FAQ match (§6.1) is the right default. When the FAQ table grows past a few thousand rows, or paraphrase misses become a real complaint, upgrade the *matcher* without touching the loop:

1. Add an `embedding vector(768)` column to `faqs` (pgvector) — or a libSQL/Turso native vector column.
2. On each FAQ insert (including `human_resolved` rows from the harvester), embed `question` and store the vector.
3. Replace `findBestFaqMatch` with a cosine-similarity search; keep the same `>= threshold` gate so the flag behaviour is unchanged.

Everything else — escalation, harvesting, the flywheel — is identical. See [`chatbotrag.md`](./chatbotrag.md) for the full embedding pipeline (chunking, task types, HNSW, backfill). The point of *this* pattern is that you ship and start learning **before** you need any of that.

---

## 14. Model Choices

| Use case | Recommended | Alternative | Why |
|---|---|---|---|
| Intent routing | `gemini-2.5-flash-lite` | `gpt-4o-mini`, `claude-haiku` | Cheapest/fastest; it's a labelling task. |
| Answer generation | `gemini-3-flash-preview` | `gpt-4o`, `claude-sonnet` | Balance of speed, cost, and grounded-answer quality. |
| FAQ match | *no model* | embeddings (§13) | Keyword scan is free and explainable at small scale. |
| Feedback / dedup checks | `gemini-2.0-flash-lite` | `gpt-4o-mini` | Best-effort, short timeout. |

Enable **prompt caching** on the (large, static) system prompt to cut cost on every answer. Note the answer path has **no failover by design** — a model error deliberately becomes a flag → escalation, which is safer than retrying into a hallucination.

---

## 15. Security Considerations

- **Auth & impersonation:** derive the user from the session token. A request may only act on the authenticated user's own thread; reject any attempt to pass another user's identity (403).
- **Prompt injection / abuse:** regex guards run before the LLM and return static refusals. This is a blocklist, not a guarantee — pair with a grounded system prompt that refuses out-of-scope requests.
- **Context scoping:** every live-lookup query filters by tenant/workspace and excludes private/hidden/soft-deleted rows. The LLM will faithfully repeat whatever you inject.
- **Human-answer trust:** `human_resolved` FAQs are staff-authored, but treat the staff channel as privileged — anyone who can reply there can teach the bot. Restrict membership; log who authored each FAQ (`sender_email` on the harvested reply).
- **Rate limiting:** cap `/bot/chat` per user; the escalation path fans out pushes and writes, so also cap flags-per-user-per-window to prevent an escalation storm.
- **Idempotency:** the `processed_replies` ledger is a correctness boundary, not an optimisation — without it the timer job duplicates FAQs and re-notifies users.

---

## 16. Implementation Checklist

### Database
- [ ] Create `chat_messages` (with `flagged`, `resolved`, `confidence`, optional feedback columns)
- [ ] Create `faqs` (with `source` column: `manual` | `human_resolved`)
- [ ] Create `escalations` linking `staff_message_id → assistant_message_id`
- [ ] Create `processed_replies` idempotency ledger
- [ ] Reserve a bot identity row + provision the staff channel with the bot as a member
- [ ] Seed initial `manual` FAQs

### Answer Pipeline
- [ ] Regex guards (abuse / injection / identity) — run **before** any LLM
- [ ] LLM intent router with enum validation + safe default
- [ ] Per-intent context builders (scoped, capped, time-stamped, text-formatted)
- [ ] Keyword `findBestFaqMatch` with a tunable threshold
- [ ] `estimateConfidence` + the `flagged` rule
- [ ] `handleChat` orchestrator: classify → log → answer → flag → log

### Human-in-the-Loop
- [ ] `escalateToStaff` — post to channel, insert `escalations`, push to staff, **fire-and-forget**
- [ ] `harvestReplies` — find replies → resolve → deliver to user + push → **INSERT INTO faqs** → mark processed
- [ ] Schedule the harvester (cron ~10 min) + a manual `/bot/admin/check-replies` trigger
- [ ] 👍/👎 feedback endpoint; 👎 re-escalates

### Frontend
- [ ] Bot appears as a pinned contact in the user's inbox
- [ ] Send / render (markdown) / receive-async-answer via history sync or push
- [ ] Feedback affordance under bot answers
- [ ] Rollout allowlist / feature flag

### Integration Points (Adapt to Your App)
- [ ] Tenant/workspace isolation on every context query
- [ ] Your push provider wired into escalation + delivery
- [ ] Your group-chat tables reused as the staff channel
- [ ] A "bot learned N answers" metric on `faqs WHERE source='human_resolved'`

---

<!-- Part of the ShipFactory feature spec library — https://github.com/vishalquantana/shipfactory -->

## About Us

We are [Quantana](https://quantana.com.au), an AI-first design and development agency working with Fortune 500s to build bespoke AI solutions and provide the audit and training needed to ensure success. [Click here to learn more](https://quantana.com.au).

## License

MIT
