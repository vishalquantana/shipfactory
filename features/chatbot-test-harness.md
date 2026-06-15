# Conversational Chatbot Test & Self-Improvement Harness

## Feature Specification for Porting

This document describes a **closed-loop testing harness for LLM chatbots** — multi-turn assistants, onboarding flows, support agents, or any "bot drives a structured conversation" product. It simulates hundreds of full conversations with an LLM playing the user, scores every transcript against objective conversation-quality metrics, visualises every chat as a live tree, then runs an automated **simulate → score → analyze → fix → re-test** loop until the bot passes a stability bar (default: **3 consecutive sets of 10 perfect conversations**).

It is provider-agnostic: the bot-under-test and the simulated user can each run on the **Anthropic API**, **OpenRouter**, or a **fully-local LMStudio** model, switched entirely by environment variables. Local Gemma runs at **$0/turn**, which is what makes "run a thousand conversations a night" affordable.

> This is the generalised, reusable version of a harness originally built to test a startup-onboarding chatbot. Names like "founder" (the simulated user) and "finalization" (the bot's success state) appear throughout — rename them to your domain (e.g. "customer" / "ticket resolved", "patient" / "triage complete").

---

## 1. Overview

The harness answers a question unit tests can't: **"Does the bot actually have good conversations with real, messy, in-character users — at scale — and is it getting better or worse with each change?"**

Core ideas:

- **An LLM plays the user.** A "user simulator" agent is given a persona, a fixed set of ground-truth facts it's allowed to state, and behavioral quirks (dumps five facts at once, asks off-topic questions, declines then reverses, states money without a currency). It drives a complete conversation against the *real* bot.
- **The real bot is under test.** No mocks of the bot's brain. The harness calls the same orchestrator/prompt code that ships to production, with the database and side-effects stubbed so no server is needed.
- **Every conversation is scored.** A small set of deterministic, LLM-free scorers turn each transcript into pass/fail signals — the headline being a single **north-star metric** (did the conversation reach its success state?).
- **A conversation-tree cache** keys replies on the exact turn-by-turn path, so re-runs are free until the bot code changes. Versioning busts the cache deliberately.
- **A live D3 tree viewer** shows every chat branching in real time, with the bot's system prompt in a panel below.
- **A self-improvement Workflow** closes the loop: it simulates a batch, mines the failures for a ranked defect backlog, dispatches a subagent to edit the real bot code, commits the change as a revertible checkpoint, bumps the version, and re-runs — repeating until the stability bar is met.

### Why both "10-then-fix" and "3×10/10"?

| Concept | What it is | Why |
|---------|------------|-----|
| **Set of 10** | One batch of 10 generated scenarios run, then scored as a unit. | Small enough to run fast and read failures by hand; large enough to surface a defect class. |
| **Fix *between* sets, never mid-batch** | Code edits happen only after a full set is scored. | Keeps each set a clean measurement of one bot version. Editing mid-batch mixes two bots in one number. |
| **3 consecutive 10/10 sets** | The convergence bar. Streak resets to 0 on any failure. | One perfect set can be luck (cache hits, easy seeds). Three in a row across fresh seeds is evidence the fix generalised, not memorised. |

---

## 2. Conversation Flow (one simulated chat)

```
                 ┌─────────────────────────────────────────────┐
                 │  Scenario JSON                              │
                 │  persona · ground_truth · behaviors ·       │
                 │  upload · expectations(must_finalize=true)  │
                 └──────────────────┬──────────────────────────┘
                                    │
   play.py init ───────────────────▼───────────────────────────
        │ stubs DB, builds REAL bot orchestrator, returns opening message
        ▼
   ┌─► USER SIMULATOR (LLM, in-character)
   │      composes next reply using ONLY ground_truth + behaviors
   │      └─ play.py step --message "<reply>"
   │             │ calls REAL bot.process_message()  ── cache lookup on turn-path
   │             ▼
   │      BOT reply  { bot_message, phase, done, turn_count }
   │      │
   └──────┘  loop until: done==true  OR  bot is wrapping up  OR  turn cap
        │
        ▼
   play.py finish ──► writes transcript JSON + runs scorers ──► scored record
                                    │
                                    ▼
                     reports/ (scorecard, trend, runs) + live tree viewer
```

The user simulator stops the moment the bot signals success (its terminal phase / "I have enough, let's wrap up"), or hits a turn cap (default 16–40 turns). Either side may run on any provider.

---

## 3. Architecture — shared core, multiple drivers

A single conversation engine is wrapped by several thin drivers so you can run one chat live for a demo, a batch for CI, or the full self-improvement loop, all hitting the same code path.

| File | Role |
|------|------|
| `play.py` | Drives **one** chat: LLM user vs the **real** bot orchestrator via `process_message`, DB-stubbed, no server. Exposes a step-mode CLI (`init` / `step` / `finish`) so an external user-simulator (e.g. a Claude Code subagent) can drive it turn-by-turn. |
| `founder.py` | The built-in simulated user (default a cheap model — Haiku or local Gemma). Renamed per domain. |
| `cache.py` | Conversation-**tree** cache keyed on the exact turn path. A HIT replays the bot reply + session state with **no** model call. Namespaced by `model` + `BOT_VERSION`. |
| `scorers.py` | Deterministic, LLM-free conversation scorers. North-star = success/finalization rate; plus re-ask, one-question-per-turn, currency-question, ends-with-question, "user had to prompt the bot". |
| `generate.py` | Deterministic matrix scenario generator (`gen-*.json`, gitignored). Hand-written `seed-*.json` are committed. |
| `run.py` | Batch runner → `scorecard.md` / `trend.jsonl` / `runs.jsonl` in `reports/`. Resumable (skips completed transcripts), rebuilds the tree live during a batch. |
| `score_batch.py` | Scores a finished batch → per-scenario PASS/FAIL + `RESULT ALL_PASS|HAS_FAILURES n/10`. |
| `tree_build.py` + `viewer.html` | Zoomable D3 tree of all chats, served over HTTP. Real-time (polls every 3s), bottom panel shows the bot system prompt, click a node → copy its path JSON. |
| `analyze.py` | Appends ranked defect classes to `todofixes.md`. |
| `archive.py` | Snapshots a failed batch's tree/scorecard/transcripts/cache before a fix, so each iteration's failing state is preserved. |
| `improve.js` | The self-improvement **Workflow** (the closed loop). See §8. |

### 3.1 The provider abstraction (the portability seam)

This is the most important file to replicate. The bot's LLM access lives behind **one function** with a stable input/output contract. Everything else — orchestrator, prompts, scorers, parser, retry/fallback machinery — is provider-agnostic.

```python
# llm_provider.py — env-gated provider selection
MODEL          = os.getenv("AI_MODEL", "claude-sonnet-4-6")
CHAT_PROVIDER  = os.getenv("CHAT_PROVIDER", "anthropic")   # "anthropic" | "openai"
OPENAI_CHAT_BASE_URL = os.getenv("OPENAI_CHAT_BASE_URL", "http://localhost:1234/v1")
OPENAI_CHAT_API_KEY  = os.getenv("OPENAI_CHAT_API_KEY", "not-needed")

def _call_llm_sync(system_prompt, messages, max_tokens, model=None):
    if CHAT_PROVIDER != "anthropic":
        return _call_openai_compat_sync(system_prompt, messages, max_tokens, model)
    # ... native Anthropic path with prompt caching + Sonnet→Haiku fallback chain
```

The OpenAI-compatible path covers **both** a local LMStudio server **and** OpenRouter — both speak the OpenAI `/chat/completions` schema, so only the transport differs. The persona prompt and the expected JSON output schema are byte-identical across providers, which is what keeps the test results comparable.

```python
def _call_openai_compat_sync(system_prompt, messages, max_tokens, model=None):
    oai_messages = [{"role": "system", "content": system_prompt}] + list(messages)
    payload = {"model": model or MODEL, "messages": oai_messages,
               "max_tokens": max_tokens, "temperature": 0.2}
    url = f"{OPENAI_CHAT_BASE_URL.rstrip('/')}/chat/completions"
    resp = httpx.post(url, json=payload,
                      headers={"Authorization": f"Bearer {OPENAI_CHAT_API_KEY}"},
                      timeout=180.0)
    if resp.status_code >= 400:          # surface LMStudio's useful 400 bodies (ctx overflow)
        raise RuntimeError(f"{resp.status_code} from {url}: {resp.text[:600]}")
    msg = resp.json()["choices"][0]["message"]
    # some local "thinking" models leave content empty → fall back to reasoning_content
    return msg.get("content") or msg.get("reasoning_content") or ""
```

> **Porting note — three gotchas that cost a day each:**
> 1. **Context window.** A real bot system prompt is large (14K+ tokens). Load the local model with a generous context (~120K); a small ctx returns opaque `400`s mid-conversation.
> 2. **"Thinking" models.** Disable reasoning/thinking mode on local models, and fall back to `reasoning_content` when `content` is empty — otherwise replies parse as blank.
> 3. **Temperature.** Pin low (`0.2`) on the bot side. High temperature is a prime cause of hallucinated/reinterpreted numbers and flaky scores.

---

## 4. Preconditions

You need exactly **one** of the three model backends below for each side (bot + user simulator). They can differ — e.g. local bot, cloud simulator.

### Option A — Local LMStudio (recommended for high-volume sweeps, **$0/turn**)

| Requirement | Detail |
|-------------|--------|
| LMStudio app | Download from lmstudio.ai. Load a chat model, e.g. `google/gemma-3-12b` (or any instruct model that fits your GPU). |
| Context length | Set **~120K** when loading the model (bot prompt is large; small ctx → 400 errors). |
| Server | Start LMStudio's local server (OpenAI-compatible) on `http://localhost:1234/v1`. |
| Verify | `curl -s -o /dev/null -w '%{http_code}' http://localhost:1234/v1/models` → `200`. |
| Thinking mode | **Disabled** in the model's generation settings. |

### Option B — OpenRouter (cloud, one key, any model)

| Requirement | Detail |
|-------------|--------|
| Account + key | `OPENROUTER_API_KEY` from openrouter.ai. |
| Base URL | `https://openrouter.ai/api/v1` (OpenAI-compatible — same code path as LMStudio). |
| Model id | Any OpenRouter model, e.g. `google/gemini-2.5-flash`, `anthropic/claude-sonnet-4-6`. |

### Option C — Anthropic direct (the default)

| Requirement | Detail |
|-------------|--------|
| Key | `CLAUDE_API` (or `ANTHROPIC_API_KEY`) in `backend/.env`. |
| Models | Primary `AI_MODEL` (e.g. `claude-sonnet-4-6`) with a `claude-haiku-4-5` fallback hop built in. |
| Prompt caching | System prompt sent with `cache_control: ephemeral` so repeated turns in a session pay ~10% of input cost. |

### Common (all options)

- Python backend with the bot's real orchestrator importable, plus `httpx`, `pydantic`, and your async runtime.
- A way to **stub the database / side-effects** so `process_message()` runs without a live DB.
- Node-capable agent runner (Claude Code / Workflow tool) **only** if you want the automated self-improvement loop in §8. The manual loop (§7) needs nothing extra.

---

## 5. Configuration

Provider selection is pure environment. Ship these as sourceable `.env` profiles so an operator switches backends with one `source`.

**`all-local-gemma.env`** — both sides local, $0, unlimited (ideal for sweeps):

```bash
# set -a; source all-local-gemma.env; set +a
CHAT_PROVIDER=openai
OPENAI_CHAT_BASE_URL=http://localhost:1234/v1
OPENAI_CHAT_API_KEY=lm-studio
AI_MODEL=google/gemma-3-12b
BOT_VERSION=v0
# user-simulator side (mirror of the above)
SIM_USER_PROVIDER=openai
SIM_USER_BASE_URL=http://localhost:1234/v1
SIM_USER_API_KEY=lm-studio
SIM_USER_MODEL=google/gemma-3-12b
```

**`bot-openrouter.env`** — bot on a cloud model, simulator wherever you like:

```bash
CHAT_PROVIDER=openai
OPENAI_CHAT_BASE_URL=https://openrouter.ai/api/v1
OPENAI_CHAT_API_KEY=${OPENROUTER_API_KEY}
AI_MODEL=google/gemini-2.5-flash
BOT_VERSION=v0
```

**Anthropic default** — set nothing; `CHAT_PROVIDER` defaults to `anthropic` and reads `CLAUDE_API`.

> `BOT_VERSION` is the **cache-busting key**. Bump it (`v0` → `v1` → …) after every bot code/prompt change so the conversation-tree cache re-tests the patched bot instead of replaying stale replies.

---

## 6. Scenario schema

Scenarios are plain JSON. Committed `seed-*.json` are hand-authored edge cases; `generate.py` produces a deterministic `gen-*.json` matrix for volume.

```json
{
  "id": "seed-002",
  "persona": { "user_name": "Ravi Kapoor", "style": "asks clarifying questions back", "english": "native" },
  "ground_truth": {
    "basic_profile": { "companyName": "NimbusAI", "sector": "SaaS", "stage": "Seed", "hq": "Bengaluru" },
    "financials":    { "revenueFY2025": 12000000, "ebitda": -3000000, "teamSize": 11 },
    "deal_intent":   { "instrument": "equity", "fundingAsk": 50000000 }
  },
  "behaviors": ["side_query_at_turn_3"],
  "upload":       { "mode": "none", "deck": null, "timing": "never" },
  "expectations": { "max_questions": 20, "max_currency_questions": 1, "must_finalize": true }
}
```

| Field | Purpose |
|-------|---------|
| `persona` | Voice/style the simulator adopts. |
| `ground_truth` | The **only** facts the simulator may state. It must never invent numbers/names. This is what lets scorers detect bot hallucination and re-asks. |
| `behaviors` | Quirks the simulator **must** exhibit — your stress tests. Examples below. |
| `upload` | Whether/when the user offers a file (for bots with document intake). |
| `expectations` | The pass bar for this scenario: `must_finalize`, caps on total questions, currency questions, etc. |

**Behavior library (rename to your domain) — each one is a known failure mode:**

| Behavior | Tests whether the bot… |
|----------|------------------------|
| `multi_fact_dump` | …extracts several facts from one message instead of re-asking each. |
| `side_query_at_turn_3` | …handles an off-topic question then returns to its flow. |
| `decline_docs_then_reverse` | …copes with "no" then later "actually, yes". |
| `currency_in_lakh` / `currency_in_crore` | …avoids needlessly asking which currency when a unit was implied. |

---

## 7. The improvement loop (manual / operator-run)

This is the loop you run by hand, and exactly what the Workflow in §8 automates.

```
1. Generate a SET of 10 fresh scenarios (new seed each time).
2. Run the batch against the current bot (BOT_VERSION=vN).
3. Score the batch  →  RESULT ALL_PASS | HAS_FAILURES n/10.
        ├─ 10/10 PASS  →  streak += 1.  If streak == 3 → DONE (converged). Else go to 1 with a new seed.
        └─ any FAIL    →  streak = 0.  Go to 4.
4. Read the FAILING transcripts. Cluster the failures into a defect class
   (re-ask, missed-finalize, double-question, lost-context, …).
5. Fix the biggest defect class by editing the REAL bot code/prompts.
6. Bump BOT_VERSION (busts the cache). Commit (revertible checkpoint).
7. Go to 1.
```

Commands (adapt paths):

```bash
# 1+2 — generate and run a set of 10
python evals/sim/generate.py --count 10 --seed 111 --out-dir /tmp/sim-v1
set -a; source evals/sim/all-local-gemma.env; set +a
export BOT_VERSION=v1
nohup python evals/sim/run.py --scenarios-dir /tmp/sim-v1 --run-id batch-v1 \
      --concurrency 4 --max-turns 40 > /tmp/sim-v1.log 2>&1 &

# 3 — score the finished set  (PASS = ALL scorers green, not just finalize)
python evals/sim/score_batch.py v1 /tmp/sim-v1     # → RESULT ALL_PASS|HAS_FAILURES n/10

# 5 — keep the bot's own unit tests green after any orchestrator edit
python -m pytest tests/sim -q
```

**Pass means *every* scorer is green, not just finalization.** A chat that finalizes but re-asked twice and asked two questions in one turn is still a FAIL — that's how the bar drives genuine conversation quality rather than just "did it reach the end".

### 7.1 The scorers (deterministic, LLM-free, fast, free)

| Scorer | Passes when | Catches |
|--------|-------------|---------|
| `finalization` (north-star) | bot reaches a terminal phase / `bot_complete` | Bot never wraps up despite having enough info. |
| `reask` | no question asked more than once | Bot forgets and re-asks a confirmed fact. |
| `one_question` | ≤1 `?` in any single bot turn | Bot crams multiple asks into one turn. |
| `ends_with_question` | every non-final, non-welcome bot turn ends with `?` | Bare acks ("No problem, we can skip that") that stall the chat. |
| `user_had_to_prompt` | user never had to nudge ("what's next?", "go on") | Bot left a turn with no forward question. |
| `currency_qs` | ≤ `max_currency_questions` (default 1) | Bot re-asks which currency after a unit was stated. |
| `follow_up_count` | ≤ `max_questions` | Excessive/runaway questioning. |

These run on the transcript JSON alone — no model call — so scoring a thousand conversations costs nothing and is fully reproducible.

---

## 8. The self-improvement Workflow (`improve.js`)

The automated version of §7, written as a multi-agent Workflow. One iteration:

| Phase | What happens |
|-------|--------------|
| **Simulate** | N user-simulator subagents drive full chats **in parallel** against the real bot via the `play.py` step-mode CLI. Each returns `{ scenario_id, finalized, defects[] }` (schema-validated). |
| **Score** | Compute the north-star rate for the iteration. If `rate >= 1.0` → **converged**, stop. |
| **Analyze** | One subagent runs `analyze.py` (appends ranked defects to `todofixes.md`), reads failing transcripts, and returns a **prioritized backlog**: `{ defect, count, root_cause, suggested_fix, target_file }`, ordered by how many chats the fix would rescue. |
| **Fix** | One subagent **archives the failed tree first** (`archive.py`), then applies the top 1–2 backlog fixes by editing the **real product code**, then `git commit`s — making each iteration a revertible checkpoint. |
| **Repeat** | `BOT_VERSION` bumps each iteration (`v0`→`v1`→…), busting the per-version cache so the next round tests the patched bot. Stops on convergence or `maxIters`. |

```js
// improve.js (sketch) — phases declared up front for progress UI
export const meta = {
  name: 'chatbot-improve-loop',
  description: 'simulate → analyze → fix (subagent) → archive → restart until all chats pass',
  phases: [{ title: 'Simulate' }, { title: 'Analyze' }, { title: 'Fix' }],
}

for (iter = 1; iter <= maxIters; iter++) {
  const version = 'v' + (iter - 1)          // cache-busting key, threaded into every CLI call

  phase('Simulate')
  const results = (await parallel(ids.map(id => () =>
    agent(userSimulatorPrompt(id, version), { schema: SIM_SCHEMA, phase: 'Simulate' })
  ))).filter(Boolean)

  const rate = results.filter(r => r.finalized).length / ids.length
  if (rate >= 1) { log('ALL CHATS PASS — converged'); break }

  phase('Analyze')
  const { backlog } = await agent(analyzePrompt(results, rate), { schema: BACKLOG_SCHEMA })

  phase('Fix')                              // archive → edit real code → commit (checkpoint)
  await agent(fixPrompt(backlog.slice(0, 2), version, iter))
}
```

> **Key discipline (baked into the prompts):** the Fix subagent makes a **minimal, targeted** edit to the named `target_file`, never a rewrite or gold-plating, and commits one checkpoint per iteration. If a fix regresses the pass rate, `git revert` a single commit. Nothing touches the operator's main checkout — run the whole loop inside an **isolated git worktree**.

The Workflow stops at "every chat finalizes". To enforce the stricter **3×10/10 all-scorers-green** bar from §1, wrap it: run the loop to convergence, then run 3 fresh sets (new seeds, full `score_batch.py`) and reset to fixing if any set isn't a clean 10/10.

---

## 9. Live tree viewer

`tree_build.py` emits a JSON tree of every conversation; `viewer.html` renders it with D3.

- **Served over HTTP** on a spare port (e.g. 8765): `python -m http.server 8765`, open `viewer.html`.
- **Real-time** — polls every 3s, so branches appear/grow live during a running batch (`run.py` rebuilds the tree itself; no separate build step).
- **Version tabs** + finalized-% + ETA + a 🎯 "follow-live" toggle that auto-pans to the newest chat.
- **Bottom panel** shows the bot's current system prompt (so you can read the thing you're tuning).
- **Click a node** → copy that conversation's path JSON (turn the live failure into a seed scenario).

Branches that share a prefix share a node, so you see at a glance where conversations diverge and where they collapse into the same failure.

---

## 10. Cost & performance: LMStudio vs OpenRouter vs Anthropic

The whole point of the provider seam is to **iterate cheaply, then validate on the model you'll ship.** Rough guidance (your hardware/prices will vary):

| Backend | $/turn | Throughput | Best for |
|---------|--------|------------|----------|
| **LMStudio (local Gemma 12B)** | **$0** | GPU-bound; one GPU serving *both* bot + simulator is slow (seconds/turn) but unlimited and private. | High-volume sweeps, overnight loops, the inner fix→re-test cycle. Run thousands of chats for free. |
| **OpenRouter (Gemini 2.5 Flash etc.)** | ~cents/conversation | High; parallelises well across the network. | Mid-loop validation and provider comparison. One key, swap models by id. |
| **Anthropic (Sonnet + Haiku fallback)** | highest | High; prompt caching cuts repeat-turn input cost ~90%. | Final validation on the production model; the default ship target. |

**Practical strategy:**

1. **Develop on local Gemma** ($0). Burn as many conversations as you want finding defect classes. The bottleneck you find here is almost always the **prompts/orchestration**, not the model — a baseline often scores 0% on *both* a frontier cloud model and local Gemma, which proves the flow is broken, not the brain.
2. **Validate mid-loop on OpenRouter** when a fix looks good — cheap, fast, and lets you compare models on the *same* scorers.
3. **Confirm the 3×10/10 bar on your production model** (Anthropic or whichever you ship), because token limits, latency, and instruction-following differ.

**Cost levers that matter (learned the hard way):**

- **The conversation-tree cache makes re-runs free** until you bump `BOT_VERSION`. Most of a loop's "runs" are cache hits.
- **Keep the cheap models for the inner loop.** Only pay frontier prices at validation gates.
- **Pin temperature low (0.2)** on the bot — high temperature both lowers scores and burns extra retries/tokens.
- **Prompt caching** (Anthropic `ephemeral`) cuts repeat-turn input cost ~90% within a session; a fallback hop to a second model pays full input cost on its first turn.
- **A local model with too small a context window silently fails (400s) mid-chat** — wasted wall-clock, not money, but the most common time sink. Size the context to your prompt + transcript.

---

## 11. Porting checklist

- [ ] **Isolate the bot's LLM access** behind one `call_llm()` function with a stable input (system prompt + messages) / output (validated JSON) contract.
- [ ] **Add the env-gated provider switch** (`CHAT_PROVIDER` → anthropic | openai-compatible) covering LMStudio + OpenRouter via the same `/chat/completions` path.
- [ ] **Stub the DB/side-effects** so the real orchestrator's `process_message()` runs serverless.
- [ ] **Write the single-chat driver** (`play.py`) with an `init`/`step`/`finish` step-mode CLI.
- [ ] **Define your success state** (the north-star "finalization" equivalent) and the terminal phases that count.
- [ ] **Write deterministic scorers** for your domain's conversation-quality defects (re-ask, multi-question, stalled turn, hallucination vs `ground_truth`).
- [ ] **Author 4–8 seed scenarios** covering your nastiest user behaviors; add a deterministic generator for volume.
- [ ] **Build the conversation-tree cache** keyed on turn-path + `model` + `BOT_VERSION`.
- [ ] **Stand up the batch runner + scorecard/trend/runs reports** (resumable).
- [ ] **(Optional) Build the live D3 tree viewer** over HTTP with the system prompt in a panel.
- [ ] **(Optional) Write the self-improvement Workflow** (simulate → analyze → fix-subagent → archive → bump version), run it inside an isolated git worktree.
- [ ] **Define your stability bar** (e.g. 3 consecutive sets of 10 with all scorers green) and wrap the loop to enforce it.
- [ ] **Wire `.env` provider profiles** so an operator switches local/cloud backends with one `source`.

---

<!-- Part of the ShipFactory feature spec library — https://github.com/vishalquantana/shipfactory -->

## About Us

We are [Quantana](https://quantana.com.au), an AI-first design and development agency working with Fortune 500s to build bespoke AI solutions and provide the audit and training needed to ensure success. [Click here to learn more](https://quantana.com.au).

## License

MIT
