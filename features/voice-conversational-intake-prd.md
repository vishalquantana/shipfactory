# Voice Conversational Intake — Product Requirements Document

> A generalised, portable pattern for building voice-powered form intake using ElevenLabs Conversational AI. This PRD is prescriptive enough that any team can implement it from scratch without guessing.

---

## 1. What This Is

A full-screen voice conversation UI where a user speaks to an AI agent ("Ella") to fill in a structured form. The agent asks questions one at a time, extracts field values via tool calls, and displays them in a live sidebar. When complete, the collected data is POSTed to an API and the user transitions to the next step.

Think of it as a **voice-powered wizard** — the agent IS the form.

---

## 2. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Browser (Client)                         │
│                                                                 │
│  ┌──────────────────────┐    WebSocket     ┌──────────────────┐ │
│  │  Voice Intake        │◄────────────────►│  ElevenLabs      │ │
│  │  Component           │  (bidirectional  │  Conversational  │ │
│  │  (.svelte/.tsx/.vue) │   audio + JSON)  │  AI Service      │ │
│  └──────────┬───────────┘                  └──────────────────┘ │
│             │                                                   │
│             │ fetch (REST)                                      │
│  ┌──────────▼───────────┐                                       │
│  │  Your App Server     │                                       │
│  │  /api/voice-token    │  ← mints signed URL                  │
│  │  /api/<resource>     │  ← saves collected data               │
│  └──────────────────────┘                                       │
└─────────────────────────────────────────────────────────────────┘
```

**Three moving parts:**
1. **ElevenLabs Agent** — hosted by ElevenLabs. You configure it via their dashboard/API. It runs an LLM, TTS, ASR, and tool-calling loop.
2. **Server Token Endpoint** — your backend mints a short-lived signed URL so the client can open a WebSocket directly to ElevenLabs.
3. **Client Component** — your frontend captures mic audio, sends it over WebSocket, receives audio + tool calls, renders the UI.

---

## 3. ElevenLabs Agent Setup

### 3.1 Create the Agent

Go to https://elevenlabs.io/app/conversational-ai → Create Agent.

| Setting | Value | Why |
|---|---|---|
| **Name** | Your agent name (e.g. "Ella") | |
| **LLM** | `gemini-2.5-flash` | Fastest response time. Alternatives: `gpt-4o-mini`, `claude-haiku`. Avoid full-size models — latency matters more than IQ for form intake. |
| **TTS Model** | `eleven_flash_v2` | Low-latency streaming voice |
| **TTS Voice** | Pick any voice from the library | |
| **Optimize Streaming Latency** | `3` (maximum) | |
| **TTS Speed** | `1.0` | |
| **ASR Quality** | `high` | |
| **ASR Provider** | `elevenlabs` | |
| **User Input Audio Format** | `pcm_16000` | Must match client AudioContext sample rate |
| **Agent Output Audio Format** | `pcm_16000` | Simplifies client playback — no decoding needed |
| **Turn Eagerness** | `eager` | Agent starts responding sooner after silence |
| **Turn Timeout** | `7.0` seconds | How long to wait before assuming user is done |
| **Max Duration** | `600` seconds (10 min) | Safety cutoff |
| **First Message** | `{{greeting}}` | Dynamic variable — set per-session from client |
| **Temperature** | `0.0` | Deterministic field extraction |

### 3.2 System Prompt Template

Replace `{{field_list}}` and `{{tool_name}}` with your actual fields and tool name.

```
IMPORTANT CONTEXT:
- User Name: {{user_name}}
- Greeting: {{greeting}}
- Continuation: {{is_continuation}}
- Known Data: {{existing_data}}
- Missing Fields: {{missing_fields}}

You are [Agent Name], a warm and professional intake assistant for [Product Name].

Your goal is to collect the following fields through natural conversation — not a form.
Ask one or two things at a time. When you learn something, call {{tool_name}}
immediately before asking the next question.

Fields to collect (ask in logical order, skip any already known):
{{field_list}}

When you have collected all required fields, say:
"Great, I've got all the information I need! [Transition message]."
Then call finish_conversation.

Tone: Warm, confident, efficient. No jargon. No filler.
Keep it brief — this should take under 2 minutes.
```

### 3.3 Client Tools to Register

Register these as **client tools** (type: `client`) in the ElevenLabs dashboard:

#### Tool 1: `save_<resource>_field`

```json
{
  "name": "save_intake_field",
  "description": "Save a single field extracted from the conversation. Call immediately when you learn a new piece of information — do not wait until the end.",
  "parameters": {
    "type": "object",
    "required": ["field", "value"],
    "properties": {
      "field": {
        "type": "string",
        "description": "Field name — one of: <comma-separated list of your field names>"
      },
      "value": {
        "type": "string",
        "description": "The extracted value as a string. For arrays use a JSON array string."
      }
    }
  }
}
```

#### Tool 2: `finish_conversation`

```json
{
  "name": "finish_conversation",
  "description": "End the conversation when all required fields have been collected. Thank the user warmly.",
  "parameters": {
    "type": "object",
    "required": [],
    "properties": {}
  }
}
```

### 3.4 Dynamic Variable Placeholders

Register these in the agent's dynamic variables config so ElevenLabs knows about them:

| Variable | Default | Purpose |
|---|---|---|
| `user_name` | `"there"` | Personalize greeting |
| `greeting` | Your default greeting | First message spoken |
| `missing_fields` | All field labels | Guides the conversation |
| `existing_data` | `"None yet"` | For continuation sessions |
| `is_continuation` | `false` | Resume vs fresh start |

### 3.5 Overrides Config

In the agent's platform settings, ensure `conversation_config_override.conversation.text_only` is set to `true` (allows client to toggle text mode). All other overrides should be `false` — the client controls behavior through `dynamic_variables`, not config overrides.

---

## 4. Database Schema

### 4.1 Generic Pattern

You need ONE table for the resource being collected. Fields that accept multiple values use JSON columns.

```sql
CREATE TABLE <resources> (
  id            TEXT PRIMARY KEY,          -- nanoid or uuid
  owner_id      TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  name          TEXT NOT NULL,
  -- Your domain fields (examples):
  field_a       TEXT,                      -- single string value
  field_b       TEXT,                      -- JSON array: '["val1","val2"]'
  field_c       TEXT,                      -- JSON array
  field_d       TEXT,                      -- single string value
  created_at    TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX idx_<resources>_owner ON <resources>(owner_id);
```

### 4.2 Drizzle ORM Example

```typescript
import { sqliteTable, text, index } from 'drizzle-orm/sqlite-core';

export const intakeRecords = sqliteTable('intake_records', {
  id:        text('id').primaryKey(),
  ownerId:   text('owner_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  name:      text('name').notNull(),
  fieldA:    text('field_a'),
  fieldB:    text('field_b', { mode: 'json' }),   // string[]
  fieldC:    text('field_c', { mode: 'json' }),   // string[]
  fieldD:    text('field_d'),
  createdAt: text('created_at').notNull().$defaultFn(() => new Date().toISOString()),
}, (t) => ({
  ownerIdx: index('idx_intake_records_owner').on(t.ownerId),
}));
```

**Important:** When using Drizzle `mode: 'json'`, NEVER `JSON.stringify()` before insert — Drizzle does it automatically. Pre-stringifying causes double-encoding and fields appear as `undefined` when read back.

---

## 5. Server Endpoints

### 5.1 Token Endpoint: `GET /api/voice-token`

Mints a short-lived signed URL for the client to open a WebSocket to ElevenLabs.

```typescript
// SvelteKit example — adapt to Express/Next/etc.
import { error, json } from '@sveltejs/kit';

export const GET = async (event) => {
  // 1. Auth check
  requireAuth(event);

  // 2. Check feature flag
  if (!ELEVENLABS_API_KEY || !ELEVENLABS_AGENT_ID) {
    throw error(503, 'Voice mode not configured');
  }

  // 3. Mint signed URL
  const url = new URL('https://api.elevenlabs.io/v1/convai/conversation/get-signed-url');
  url.searchParams.set('agent_id', ELEVENLABS_AGENT_ID);

  const res = await fetch(url.toString(), {
    method: 'GET',
    headers: { 'xi-api-key': ELEVENLABS_API_KEY }
  });

  if (!res.ok) {
    const body = await res.text().catch(() => '');
    throw error(502, `Failed to mint signed URL: ${body}`);
  }

  const data = await res.json();
  return json({ signedUrl: data.signed_url });
};
```

**Why `get-signed-url` and not `/token`?** The signed URL grants full WebSocket access for bidirectional audio. The `/token` endpoint is for WebRTC sessions and doesn't reliably work with raw WebSocket connections.

### 5.2 Resource Endpoint: `POST /api/<resource>`

Receives the collected fields and creates the record.

```typescript
export const POST = async (event) => {
  requireAuth(event);

  const body = await event.request.json();

  // Validate with Zod — MUST include every field the client sends
  const parsed = schema.parse(body);

  const id = nanoid();
  await db.insert(intakeRecords).values({
    id,
    ownerId: event.locals.session.userId,
    ...parsed,
  });

  return json({ id });
};
```

---

## 6. Client Component

### 6.1 Screen Layout

```
┌─────────────────────────────────────────────────────────────────────┐
│ ◀ Back                                                         [X] │
├──────────────────────────────────┬──────────────────────────────────┤
│                                  │  Resource Being Built            │
│                                  │  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░  3 / 6   │
│                                  │                                  │
│         ┌─────────┐              │  FIELD A                         │
│         │         │              │  Acme Insurance Corp             │
│         │   ORB   │              │                                  │
│         │         │              │  FIELD B                         │
│         └─────────┘              │  General Liability, Property     │
│          LISTENING               │                                  │
│                                  │  FIELD C                         │
│      Ella is listening           │  Waiting...                      │
│                                  │                                  │
│        [ ‖ Pause ]               │  FIELD D                         │
│                                  │  Waiting...                      │
│  ─────────────────────────────── │                                  │
│  ELLA                            │  FIELD E                         │
│  Hi! Let me help you set up...   │  Waiting...                      │
│                                  │                                  │
│                           YOU    │  FIELD F                         │
│             It's called Acme...  │  Waiting...                      │
│                                  │                                  │
│  ELLA                            │                                  │
│  Great name! What lines of...    │                                  │
├──────────────────────────────────┴──────────────────────────────────┤
│                    (full-screen overlay, z-index: 150)              │
└─────────────────────────────────────────────────────────────────────┘
```

**Layout rules:**
- Full-screen overlay (fixed, inset: 0) with backdrop blur
- Left panel (60%): animated orb, status, pause button, transcript
- Right panel (40%): field progress bar, live field values, completion hint
- Transcript scrolls, auto-scrolls to bottom on new entries
- Fields highlight with accent left-border when filled

### 6.2 Finishing State

When `finish_conversation` fires, DON'T navigate immediately. Show a transition card:

```
┌──────────────────────────────────┐
│                                  │
│            ✓ (green circle)      │
│                                  │
│        Basics captured           │
│                                  │
│  Ella has collected the key      │
│  details for Acme Insurance.     │
│  Next, the wizard will walk      │
│  you through the rest.           │
│                                  │
│   [ Continue to next step → ]    │
│                                  │
└──────────────────────────────────┘
```

### 6.3 Component State Machine

```
                 ┌──────────┐
     onMount ──► │connecting│
                 └────┬─────┘
                      │ WS open + conversation_initiation_metadata
                      ▼
                 ┌──────────┐
         ┌──────│   idle    │◄─────────────────────────┐
         │      └────┬─────┘                            │
         │           │ user speaks (VAD/audio)          │
         │           ▼                                  │
         │      ┌──────────┐                            │
         │      │listening │──── agent audio ───►┌──────────┐
         │      └──────────┘                     │speaking  │
         │           ▲                           └────┬─────┘
         │           │ pendingChunks === 0            │
         │           └────────────────────────────────┘
         │
         │ finish_conversation tool call
         ▼
    ┌──────────┐
    │finishing │──── user clicks "Continue" ───► oncomplete(id)
    └──────────┘

    Any state ──── WS error ───► ┌──────────┐
                                 │  error   │──── retry ───► connecting
                                 └──────────┘

    Any state ──── pause ───► cleanup() ──── resume ───► connecting
```

### 6.4 WebSocket Protocol (Client-Side Implementation)

#### Connection Setup

```typescript
// 1. Create AudioContext (16kHz to match ElevenLabs)
audioCtx = new AudioContext({ sampleRate: 16000 });
if (audioCtx.state !== 'running') await audioCtx.resume();

// 2. Fetch signed URL from YOUR server
const { signedUrl } = await fetch('/api/voice-token').then(r => r.json());

// 3. Open WebSocket directly to ElevenLabs
ws = new WebSocket(signedUrl);
```

#### On WebSocket Open — Send Init Data

```typescript
ws.addEventListener('open', () => {
  ws.send(JSON.stringify({
    type: 'conversation_initiation_client_data',
    dynamic_variables: {
      mode: 'your-mode',
      user_name: userName,
      greeting: 'Hi! I\'m here to help you...',
      missing_fields: 'Field A, Field B, Field C',
      existing_data: 'None yet',
      is_continuation: false
    }
  }));
});
```

**DO NOT use `conversation_config_override`** unless the agent has those overrides explicitly enabled in the dashboard. Sending unsupported overrides silently causes the WebSocket to close.

#### Message Types to Handle

| Type | What it is | What to do |
|---|---|---|
| `conversation_initiation_metadata` | Server confirms session started | Extract `agent_output_audio_format`, set status to `idle` |
| `audio` | Agent speech chunk (base64 PCM) | Decode, schedule playback, set status to `speaking` |
| `agent_response` | Agent speech as text | Append to transcript |
| `user_transcript` | Your speech as text | Append to transcript |
| `client_tool_call` | Agent wants to save a field | Extract field/value, update UI, send `client_tool_result` |
| `interruption` | User interrupted agent | Cancel pending audio, set status to `listening` |
| `vad_score` | Voice activity detection score | Optional: log when > 0.5 |
| `ping` | Keep-alive | **MUST respond with pong** or connection drops |

#### Ping/Pong (CRITICAL)

```typescript
if (type === 'ping') {
  const pingEvent = msg.ping_event as Record<string, unknown> | undefined;
  ws.send(JSON.stringify({
    type: 'pong',
    event_id: pingEvent?.event_id ?? msg.event_id
  }));
}
```

**If you don't respond to pings, ElevenLabs drops the WebSocket.** This is the #1 cause of "WebSocket closed immediately after init."

#### Sending Tool Results

When a `client_tool_call` arrives, you MUST send a result back:

```typescript
ws.send(JSON.stringify({
  type: 'client_tool_result',
  tool_call_id: toolCallId,
  result: 'field saved',     // or any string
  is_error: false
}));
```

#### Sending Mic Audio

```typescript
// Use ScriptProcessorNode (deprecated but most portable)
const scriptProcessor = audioCtx.createScriptProcessor(2048, 1, 1);
source.connect(scriptProcessor);
scriptProcessor.connect(audioCtx.destination);

scriptProcessor.onaudioprocess = (e) => {
  if (!ws || ws.readyState !== WebSocket.OPEN) return;
  const float32 = e.inputBuffer.getChannelData(0);

  // Convert Float32 → Int16 PCM
  const int16 = new Int16Array(float32.length);
  for (let i = 0; i < float32.length; i++) {
    int16[i] = Math.max(-32768, Math.min(32767, float32[i] * 32768));
  }

  // Base64 encode and send
  const bytes = new Uint8Array(int16.buffer);
  let b64 = '';
  const CHUNK = 8192;
  for (let i = 0; i < bytes.length; i += CHUNK) {
    b64 += String.fromCharCode(...bytes.subarray(i, i + CHUNK));
  }
  ws.send(JSON.stringify({ user_audio_chunk: btoa(b64) }));
};
```

**Buffer size 2048** (not 4096) = 128ms chunks at 16kHz. Smaller = lower latency.

#### Playing Agent Audio

```typescript
async function playAudioChunk(audioBase64: string) {
  const sampleRate = parseInt(outputFormat.replace('pcm_', ''), 10) || 16000;
  const bytes = Uint8Array.from(atob(audioBase64), c => c.charCodeAt(0));
  const int16 = new Int16Array(bytes.buffer);
  const float32 = new Float32Array(int16.length);
  for (let i = 0; i < int16.length; i++) {
    float32[i] = int16[i] / 32768;
  }

  const audioBuffer = audioCtx.createBuffer(1, float32.length, sampleRate);
  audioBuffer.copyToChannel(float32, 0);

  const source = audioCtx.createBufferSource();
  source.buffer = audioBuffer;
  source.connect(audioCtx.destination);

  // SCHEDULE sequentially — without this, chunks overlap (sounds fast then slow)
  const now = audioCtx.currentTime;
  if (nextStartTime < now) nextStartTime = now;
  source.start(nextStartTime);
  nextStartTime += audioBuffer.duration;
}
```

**You MUST schedule chunks sequentially** using `nextStartTime`. Without this, audio overlaps at the beginning and gaps appear later. Reset `nextStartTime = 0` on interruption events.

---

## 7. Environment Variables

| Variable | Example | Where |
|---|---|---|
| `ELEVENLABS_API_KEY` | `sk_...` | Server `.env` |
| `ELEVENLABS_AGENT_ID` | `agent_...` | Server `.env` |

**Feature flag pattern:**

```typescript
export const voiceEnabled = Boolean(ELEVENLABS_API_KEY) && Boolean(ELEVENLABS_AGENT_ID);
```

Show/hide the voice option in the UI based on this flag. Return 503 from the token endpoint if not configured.

---

## 8. Common Pitfalls & Fixes

| Symptom | Cause | Fix |
|---|---|---|
| WebSocket closes immediately after init | Not responding to `ping` messages | Implement pong handler (Section 6.4) |
| WebSocket closes immediately after init | Sending `conversation_config_override` when overrides are disabled on agent | Only use `dynamic_variables`, not config overrides |
| Agent speaks but doesn't hear you | Mic starts after WS closes | Acquire mic before or immediately after WS open (within 100ms) |
| Audio plays too fast then gaps appear | Chunks not scheduled sequentially | Use `nextStartTime` pattern (Section 6.4) |
| `AudioContext` won't start | Browser autoplay policy | Create AudioContext inside user gesture (button click) |
| Fields arrive as `undefined` | Drizzle `mode:'json'` + manual `JSON.stringify()` | Never stringify before insert with Drizzle json mode |
| Agent returns wrong tool name | Agent prompt has tools for multiple modes | Check both tool names (e.g. `save_profile_field` OR `save_intake_field`) |
| Token endpoint returns 502 | Wrong ElevenLabs API endpoint | Use `get-signed-url`, NOT `/token` (which is for WebRTC) |
| Console shows `ScriptProcessorNode deprecated` | Expected — it's the most portable approach | Ignore warning, or migrate to AudioWorklet for production |

---

## 9. Checklist: Adding Voice Intake to a New Resource

Use this as a step-by-step implementation checklist:

### ElevenLabs Dashboard
- [ ] Create agent with settings from Section 3.1
- [ ] Paste system prompt from Section 3.2, customized with your fields
- [ ] Register `save_<resource>_field` client tool with your field names
- [ ] Register `finish_conversation` client tool
- [ ] Add dynamic variable placeholders (Section 3.4)
- [ ] Set turn eagerness to `eager`
- [ ] Test in ElevenLabs playground first

### Server
- [ ] Add `ELEVENLABS_API_KEY` and `ELEVENLABS_AGENT_ID` to `.env`
- [ ] Create feature flag (`voiceEnabled`)
- [ ] Create `GET /api/voice-token` endpoint (Section 5.1)
- [ ] Create `POST /api/<resource>` endpoint (Section 5.2)
- [ ] Ensure Zod schema includes ALL fields the client sends

### Database
- [ ] Create table with JSON columns for array fields (Section 4)
- [ ] Run migration
- [ ] Test insert/read round-trip

### Client Component
- [ ] Create voice intake component with state machine (Section 6.3)
- [ ] Implement WebSocket connection with signed URL
- [ ] Send `conversation_initiation_client_data` on open (Section 6.4)
- [ ] Handle ALL message types (Section 6.4 table)
- [ ] **Implement ping/pong handler** (most forgotten step)
- [ ] Implement mic capture at 16kHz with 2048 buffer
- [ ] Implement audio playback with sequential scheduling
- [ ] Implement `client_tool_call` → field update → `client_tool_result` loop
- [ ] Add transcript display with auto-scroll
- [ ] Add pause/resume button
- [ ] Add finishing transition card (don't navigate immediately)
- [ ] Add retry button for error state
- [ ] Add orb animation component (or simple status indicator)
- [ ] Wire up `finish_conversation` → POST to server → show transition card

### Integration
- [ ] Add mode picker card (form vs AI vs voice)
- [ ] Gate voice card behind `voiceEnabled` flag
- [ ] Full-screen overlay with backdrop blur
- [ ] Close button on overlay
- [ ] Test on Chrome, Safari, Firefox (mic permissions differ)
- [ ] Test with slow network (WebSocket reconnection)

---

## 10. API Reference Quick-Reference

### ElevenLabs REST API

**Mint signed URL:**
```
GET https://api.elevenlabs.io/v1/convai/conversation/get-signed-url?agent_id={agent_id}
Headers: xi-api-key: {api_key}
Response: { "signed_url": "wss://..." }
```

**Get agent config:**
```
GET https://api.elevenlabs.io/v1/convai/agents/{agent_id}
Headers: xi-api-key: {api_key}
```

**Update agent:**
```
PATCH https://api.elevenlabs.io/v1/convai/agents/{agent_id}
Headers: xi-api-key: {api_key}, Content-Type: application/json
Body: { "conversation_config": { ... } }
```

### ElevenLabs WebSocket Messages (Client → Server)

| Message | Format |
|---|---|
| Init | `{ "type": "conversation_initiation_client_data", "dynamic_variables": {...} }` |
| Audio | `{ "user_audio_chunk": "<base64 PCM>" }` |
| Tool result | `{ "type": "client_tool_result", "tool_call_id": "...", "result": "...", "is_error": false }` |
| Pong | `{ "type": "pong", "event_id": <from ping> }` |

### ElevenLabs WebSocket Messages (Server → Client)

| Message | Key fields |
|---|---|
| `conversation_initiation_metadata` | `.conversation_initiation_metadata_event.agent_output_audio_format` |
| `audio` | `.audio_event.audio_base_64` |
| `agent_response` | `.agent_response_event.agent_response` |
| `user_transcript` | `.user_transcription_event.user_transcript` |
| `client_tool_call` | `.client_tool_call.tool_name`, `.client_tool_call.tool_call_id`, `.client_tool_call.parameters` |
| `interruption` | (no payload — cancel pending audio) |
| `ping` | `.ping_event.event_id` |
| `vad_score` | `.vad_score_event.vad_score` (0.0–1.0) |

---

## 11. Recommended Agent Settings Comparison

| Setting | Fast (recommended) | Quality | Notes |
|---|---|---|---|
| LLM | `gemini-2.5-flash` | `gpt-4o` / `claude-sonnet` | Flash is 2-3x faster |
| TTS | `eleven_flash_v2` | `eleven_multilingual_v2` | Flash trades slight quality for speed |
| Streaming Latency | `3` | `1` | Higher = more aggressive streaming |
| Turn Eagerness | `eager` | `patient` | Eager responds faster but may cut off slow speakers |
| Temperature | `0.0` | `0.3` | 0 = deterministic extraction |
| Buffer Size | `2048` | `4096` | Smaller = lower latency, higher CPU |

---

## 12. Security Considerations

- **Never expose `ELEVENLABS_API_KEY` to the client.** The signed URL is the client's credential — it's short-lived and scoped to one conversation.
- **Always authenticate the token endpoint.** Anyone who can call `/api/voice-token` can start a billable ElevenLabs conversation.
- **Validate collected data server-side.** The voice agent extracts strings — your server must validate them with the same rigor as form input (Zod schema, type coercion, length limits).
- **Rate-limit the token endpoint.** Each signed URL costs money. Apply per-user rate limiting.
- **Don't trust tool call parameters.** The LLM generates them — they may contain hallucinated field names or malformed values. Whitelist expected field names.

---

<!-- Part of the ShipFactory feature spec library — https://github.com/vishalquantana/shipfactory -->

## About Us

We are [Quantana](https://quantana.com.au), an AI-first design and development agency working with Fortune 500s to build bespoke AI solutions and provide the audit and training needed to ensure success. [Click here to learn more](https://quantana.com.au).

## License

MIT
