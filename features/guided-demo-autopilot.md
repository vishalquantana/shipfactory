# Guided Demo Autopilot (narrated, self-driving product tour)

## Feature Specification for Porting

This document describes an **autopilot product tour**: press Play and the real application drives itself. A visible cursor glides along a curved path to each control, a spotlight dims everything else, a voice narrates each beat while subtitles reveal word by word, real clicks happen, and pages change. The tour pauses the instant the viewer touches anything, and resumes on the same page and step. It runs on a phone, a tablet or a laptop, offline, with pre-recorded studio-quality narration, and degrades gracefully to browser speech and then to subtitles only.

It is **not a video and not a mock**. Every click goes to the real app, so the demo can never drift from the product, and a viewer can take over at any moment ("let me try that").

> Reference build: a React 19 + Vite + Tailwind + `motion` PWA (a B2B reinsurance platform demo, Sep 2026). Two tours, 47 + 41 steps, 79 narrated lines, ~6,100 characters of ElevenLabs audio (~7 min), about 4.5 min per tour at 1×. Built and hardened in a few days with Claude Code. The code in §12 is lifted from that build and is unit- and e2e-tested there.

Everything needed to rebuild it is here: the UX, the architecture, every algorithm and timing constant, the tested core code, the narration pipeline, the test harness, the lessons from 13 rounds of fixes, and the **prompts** (§14) that make an AI coding agent build it end to end.

---

## 1. Overview

| Capability | What the viewer experiences |
|---|---|
| **Self-driving tour** | An ordered script of steps; each step navigates, finds a real element, scrolls it into comfortable view, glides the cursor there, optionally clicks, and narrates one line |
| **Spotlight** | Everything except the target dims; a soft ring springs from target to target and follows it through layout shifts |
| **Cursor** | A large branded pointer with a breathing halo moves on a curved, spring-eased path; a press plus a double-ring ripple on click |
| **Narration** | Pre-rendered MP3 per line (two voices) → browser speech synthesis → silent timer. Subtitles reveal word by word in sync |
| **Subtitle dock** | Frosted-glass bar, at most 2 lines (3 on a phone), bottom of the screen; jumps to the top only if it would cover the target |
| **Controls** | Play/pause, previous/next step, chapter overview, voice menu (voice, subtitles-only, replay line), speed 1× / 1.25×, seat (persona) switch, exit. Auto-hide after 2 s idle |
| **Interruptible** | Any real tap or keypress pauses. Resume returns to the same seat, page and step. Esc exits |
| **Chapters** | A full-screen grid of chapter cards with thumbnails. Jumping to a chapter resets the data and fast-forwards state, so any chapter plays cold |
| **Up next** | Between chapters, a 3 s countdown card ("Stay here" / "Go now") |
| **Multiple tours** | E.g. "the owner's view" and an "end-to-end lifecycle" that hops between personas, with a colour-wash seat title card at each switch |
| **Deep links** | `?play_demo=true` autostarts (no splash); `&tour=`, `&chapter=`, `&speed=`, `&voice=`, `&mute=1`. `?demo=1` shows the splash |
| **Autoplay-safe** | If the browser blocks sound, the tour keeps running on subtitles at the recorded pace and shows a pulsing **Tap for sound** pill. The first tap is consumed (no pause, no click-through) and replays the line with audio |
| **Offline** | As a PWA, everything including the audio is precached; Safari range requests are served from cache; app updates never reload mid-tour |
| **Testable** | The overlay exposes its state as `data-autopilot-*` attributes and a timing log; an e2e harness plays the whole tour on phone and desktop and fails on skipped steps, slow targets, or dead air over 1.2 s |

**States:** `idle → splash → running ⇄ paused → finished` (Esc from any → `idle`).

---

## 2. User Flow

### 2.1 Starting

```
User menu "Play guided demo"          ?demo=1                    ?play_demo=true[&tour=&chapter=&speed=&voice=&mute=1]
  (a click = user gesture)              │                            │  (no gesture: audio may be blocked)
        │                               ▼                            │
        │                     ┌──────────────────────┐               │
        │                     │ Splash                │               │
        │                     │  Tour tabs            │               │
        │                     │  Voice picker         │               │
        │                     │  [ ▶ Play ]  (gesture) │               │
        │                     └──────────┬───────────┘               │
        ▼                                ▼                            ▼
  unlockMedia() synchronously in the click ─┐          strip params from the URL (reload won't restart)
                                            ▼                            │
                   reset demo data → primary persona → navigate('/') ◄──┘
                                            │
                                            ▼
                                       runFrom(0)
```

### 2.2 One step (the heartbeat)

```
playStep(i)
 ├─ preload audio for step i+1
 ├─ skip if a step it `requires` was skipped
 ├─ seat change?  → seat title card (380 ms) → setPersona → navigate → hold (1220 ms) → card out
 ├─ route?        → navigate, wait 480 ms
 ├─ page?         → wait until pathname matches, ≥320 ms since navigation, and the outgoing page has unmounted
 ├─ target?       → waitForTarget (MutationObserver + 120 ms poll, timeout 2.5 s)
 │                   not found → fallbackRoute | fallback() | mark skipped & return
 ├─ ▶ START NARRATION NOW (line plays *under* the movement: no dead air)
 ├─ spotlight → target;  scroll into the comfort zone (clear of top bar + dock), wait until still
 ├─ action ≠ none/wait → cursor glides along a curve (0.6–0.9 s)
 ├─ action = click → wait clickDelay (650 ms) → re-aim if the page moved → press + ripple + real click
 │                    → if something opened on top of the target, drop the spotlight
 ├─ await narration end
 └─ dwell (600 ms default) with a draining progress line
```

### 2.3 Interrupting

```
Real pointerdown / key (isTrusted)  →  pause: abort the run, stop audio, show the full line, remember the page
Space / k                           →  toggle pause/resume
← / →                               →  previous / next step (→ still performs this step's click so later steps find their state)
Esc                                 →  exit: stop, clear the overlay, hand the app back in the primary seat
Resume                              →  restore the seat this step needs → return to the remembered page → runFrom(index)
Chapter card                        →  reset data → prepare(actions) fast-forward → navigate → run from the chapter's first step
```

---

## 3. Architecture

```
src/demo/
  types.ts          # DemoStep, DemoChapter, DemoTour, FindText, DemoAction   (§12.1)
  runner.ts         # pure: target resolution, waits, timing math, voice pick (§12.2), unit-tested
  narrator.ts       # narrator choice (persisted) + audio path per step           (§12.3)
  start.ts          # startGuidedDemo(): window event, called inside a click       (§12.4)
  script.ts         # Tour A steps + NARRATION export + CHAPTER_META               (pure data)
  lifecycle.ts      # Tour B steps (multi-persona) + its NARRATION + CHAPTER_META
  chapters.ts       # TOURS registry, chapter prepare() fast-forwards, seatAt()
  Autopilot.tsx     # the controller: state machine, playStep, narrate, cursor, scroll, input handling,
                    #   portal with all overlay UI, SubtitleDock + dockEdge()       (§5, §12.5–12.8)
  panels.tsx        # ChapterOverview, UpNextCard, ProgressRail, NarratorMenu, SeatMenu, TourTabs
  overlay/
    tokens.ts       # colours, z-index (2147483000), Rect type
    Spotlight.tsx   # one div + a giant box-shadow (§12.6)
    Cursor.tsx      # pointer SVG + halo + ripple
    Splash.tsx      # tour tabs, voice picker, Play
    SeatCard.tsx    # full-screen colour-wash "Now in <persona>'s seat"
    EndCard.tsx     # finale lines + Replay / Close
  runner.test.ts, dock.test.ts
scripts/
  gen-narration.mjs     # ElevenLabs → public/audio/demo/<voice>/<stepId>.mp3 + manifest.json (§7, §12.9)
  autopilot-e2e.mjs     # Playwright: full tours, chapter jumps, autoplay policy, audio (§9.3)
  capture-thumbs.mjs    # chapter thumbnails from a muted 2× run → public/demo/thumbs/<chapter>.webp
public/
  audio/demo/<voice>/<id>.mp3, manifest.json
  demo/thumbs/<chapter>.webp
  sw-range.js           # (PWA) answers Range requests for precached mp3 with 206 (§12.11)
```

Mounted once inside the app shell (inside the router and the store provider): `<Autopilot />`. Started from anywhere with `startGuidedDemo()`.

**Data flow.** The script is *data*. The controller reads it, drives the router and the store (`navigate`, `actions.reset`, `actions.setPersona`, and the chapter `prepare` actions), and clicks real DOM elements. The overlay is a portal at the top of the z-order, marked `data-autopilot-ui`, so target lookups and the e2e harness can tell it apart from the app.

**Why a long-running async loop and refs.** The tour is one `async for` loop over steps, cancelled with an `AbortController`. React state is stale inside a long loop, so every value the loop needs (seat, muted, narrator, speed, reduced motion, pathname, dock height, "blocked") lives in a `live` ref, refreshed in a `useLayoutEffect` on every render. Steps read `tourRef.current` so a tour switch applies immediately. Every `sleep()` takes the abort signal, so pause and exit are instant.

---

## 4. The Step Script

### 4.1 Step fields (see §12.1 for the type)

| Field | Purpose |
|---|---|
| `id` | Stable id. Also the audio filename and the thumbnail key. **Never reuse or rename casually** (it orphans audio) |
| `chapter` | Chapter title; changes trigger "Up next" |
| `seat?` | Persona to play this step in (title card → setPersona → navigate). Later steps inherit it |
| `route?` | Navigate here first (if not already there) |
| `page?` | Path prefix the target belongs to: don't look for it until this page has settled |
| `target?` | CSS selector(s), tried in order. Prefer `[data-demo="…"]` |
| `findText?` | Fallback: `{ role: 'button'|'link'|'tab'|'heading'|'any', text: string|RegExp }` on accessible text |
| `closest?` | Widen the found element to an ancestor (spotlight the whole card, not its heading) |
| `action` | `click` · `hover` (cursor rests, no click) · `scroll` (spotlight, no cursor) · `wait` · `none` (line only) |
| `subtitle` | The spoken line (also the subtitle) |
| `dwellMs?` | Hold after the line (default 600). Use ~1000 after a visible state change and ~3500 on the finale |
| `clickDelayMs?` | Cursor lands → click (default 650). 900 when the viewer should read the button first |
| `timeoutMs?` | Wait for the target (default 2500). 700 for nav links that may be hidden on a phone; 9000 for long in-app animations |
| `deliberate?` | Reason this step waits on purpose (e.g. "extraction animation ~3.9 s"); excluded from the gap metric. The screen **must show progress** meanwhile |
| `requires?` | Skip if the named step was skipped (follow-ups to an optional control) |
| `fallbackRoute?` | Target not found (nav hidden in a phone's "More" menu) → just navigate there |
| `fallback?` | Target not found → run this (e.g. dispatch an app event that opens the panel) |
| `before?` | Run first (e.g. dispatch Escape to close leftover sheets before the finale) |
| `audioSrc?` | Override the audio path |

### 4.2 Example (generic)

```ts
const nav = (href: string) => [
  `[data-demo="nav-${href.split('/')[1] || 'home'}"]`,
  `nav[aria-label="Primary"] a[href="${href}"]`,
  `aside a[href="${href}"]`,
  `nav a[href="${href}"]`,
]

const STEPS: DemoStep[] = [
  { id: 'intro', chapter: 'Dashboard', route: '/', action: 'none',
    subtitle: 'This is Acme. One deal, from a messy upload to cash reconciled.' },
  { id: 'kpis', page: '/', chapter: 'Dashboard', target: '[data-demo="kpis"]', closest: 'section',
    action: 'scroll', subtitle: 'Targets against actuals, every morning.' },
  { id: 'nav-pipeline', chapter: 'Pipeline', target: nav('/pipeline'), timeoutMs: 700,
    fallbackRoute: '/pipeline', action: 'click', subtitle: 'Your pipeline, in your words.' },
  { id: 'open-deal', page: '/pipeline', chapter: 'Pipeline', target: '[data-demo="deal-northwind"]',
    findText: { role: 'button', text: /Northwind/ }, action: 'click',
    subtitle: 'Every conversation is logged against the account.' },
  { id: 'advance', page: '/pipeline', chapter: 'Pipeline', findText: { role: 'button', text: /^Advance to/ },
    action: 'click', clickDelayMs: 900, dwellMs: 1000,
    subtitle: 'One tap moves it forward, and the whole team sees it happen.' },
  { id: 'upload', page: '/intake', chapter: 'Intake', findText: { role: 'button', text: /^Use sample/ },
    action: 'click', clickDelayMs: 500, dwellMs: 1000, subtitle: 'The customer sends its pack, in whatever format.' },
  { id: 'extraction', page: '/intake', chapter: 'Intake', target: '[data-demo="extracted-fields"]', closest: 'section',
    deliberate: 'waits for the extraction animation (~3.9 s after Use sample)', timeoutMs: 9000,
    action: 'scroll', dwellMs: 1000, subtitle: 'Sixteen fields in seconds, and it’s honest about the two it isn’t sure of.' },
  { id: 'finale', chapter: 'Finale', before: closeOverlays, route: '/', action: 'none', dwellMs: 3500,
    subtitle: 'Owned by you. Live in January.' },
]

export const DEMO_SCRIPT = STEPS
/** Single source of truth for the audio generator (Node imports this .ts file directly). */
export const NARRATION = STEPS.map(({ id, subtitle }) => ({ id, subtitle }))
export const CHAPTER_META = [
  { id: 'dashboard', title: 'Dashboard', thumbStep: 'kpis' },
  { id: 'pipeline', title: 'Pipeline', thumbStep: 'advance' },
  { id: 'intake', title: 'Intake', thumbStep: 'extraction' },
  { id: 'finale', title: 'Finale', thumbStep: 'finale' },
]
```

Keep script files **free of runtime imports** (type-only imports), so `node scripts/gen-narration.mjs` can `import()` them directly (Node ≥ 22.6 strips TypeScript types natively; Node 23.6+ does it without a flag).

### 4.3 Writing the lines (the part that makes people love it)

- **One idea per step, two lines max** (~12–22 words). Spoken aloud, so plain words, short sentences, contractions.
- **Use the customer's own vocabulary and phrases.** Mine call transcripts for their exact words ("kit the submission", "suspects") and put them in the lines. Hearing their own words is what sells it.
- **Say the outcome, not the UI.** "One tap moves it into vetting, and the whole team sees it" beats "Click the Advance button".
- **Concrete numbers** that are visible on screen ("47% against a 55% target").
- **The action happens during the line.** The line starts before the cursor moves, so write lines that are true *while* the click happens.
- **Name the human guardrail** wherever AI acts: "Nothing goes out until you approve it."
- **Chapters of 3–6 steps**, 8–11 chapters per tour, ~4–5 min per tour at 1×. A finale of one or two lines, then the end card.
- **Spell numbers the way they should be spoken** if the TTS stumbles ("ninety percent", "1 January").

---

## 5. Engine Algorithms

### 5.1 Target resolution (§12.2)
- Selectors in order; the first **visible** match wins (rendered, has client rects, not `visibility:hidden`/`display:none`), so the hidden phone tab bar on a desktop is ignored.
- Never match inside the overlay (`[data-autopilot-ui]`).
- Text fallback by role (button/link/tab/heading), or `any` = the innermost element whose own text matches (TreeWalker over text nodes).
- `closest` widens the element after it is found.
- **Wait** with a `MutationObserver` *and* a 120 ms poll (layout and visibility changes don't always mutate the DOM). On timeout: `console.warn` and skip.

### 5.2 Page settling (the #1 source of flaky tours)
Route transitions with exit animations keep the **outgoing page mounted** for a moment, still rendering the *new* route's content. Clicking into it opens a sheet that is then destroyed when the page remounts. So, for a step with `page`:
1. Wait (≤2.5 s) until `pathname` matches `page`.
2. Wait until ≥320 ms since the last route change.
3. Wait (≤2.5 s) until the element that was `main > :first-child` at navigation time is gone or replaced. After 400 ms, nudge the shell to re-render with a same-URL `navigate(..., { replace: true, state: { flush: Date.now() } })`; some devices don't flush the finished exit otherwise.
4. If any of that took more than 60 ms, settle another 200 ms.

Also make exiting pages ignore taps (`pointer-events: none` while `!isPresent`).

### 5.3 Narration (fallback chain, §12.5)
```
muted or blocked?            → timer
audio file for this voice?   → <audio>.play()
                                 ├─ ended          → done
                                 ├─ error          → speech
                                 └─ NotAllowedError → BLOCKED: set needsGesture, timer at the *recorded* duration (manifest)
otherwise                    → speechSynthesis (best voice, §12.2 pickVoice)
                                 ├─ onend → done; onerror 'not-allowed' → BLOCKED
                                 └─ no onstart within 1.2 s → cancel → timer
timer                        → words × 380 ms / speed  (or the manifest duration when blocked)
safety guard                 → always resolve after fallback × 2.5 + 4 s
```
- **Word reveal:** interval = `max(60, totalMs × 0.82 / words)` (finish slightly before the voice). For audio, start on `loadedmetadata` using `duration / playbackRate`. For speech, also bump on `onboundary` word events. Reduced motion shows the whole line at once.
- **Speed** sets `audio.playbackRate`, `utterance.rate`, and divides every wait.
- **Preload** the next step's MP3 with `fetch()` while the current one plays.
- **The narrator is read per line**, so changing voice mid-tour applies from the next line; "Subtitles only" stops the current line immediately.
- **Audio unlock:** inside the user's click, synchronously, `play()` the first file **muted**, then pause, unmute and rewind; also `speak()` a zero-volume space. Without this, iOS/Safari refuse every later `play()`.

### 5.4 Cursor (§12.7)
- Landing point: small targets → centre; big targets (>280×120) → 62% across and ≤110 px down (look at the content, not the corner). Clamp inside the viewport (top ≥72 px for the sticky bar). If the point falls on the subtitle dock, slide it above or below the dock within the target.
- Path: a **quadratic Bézier** whose control point is pushed perpendicular to the straight line by `min(140, dist × 0.22)`, always bowing the same way (feels hand-moved). Animate t 0→1 with a spring (bounce 0.16), duration `clamp(0.45 + dist/1800, 0.6, 0.9) s / speed`. Reduced motion: jump.
- At rest: the halo breathes (scale 1→1.16, 2.2 s loop).
- Click: `pointerdown` → 110 ms → `pointerup` + `el.click()`, with a press state, a double-ring ripple and a spotlight flash. Before clicking, re-resolve if the element detached and re-glide if it moved >6 px.
- After the click, wait ~500 ms; if `elementFromPoint` at the target's centre is no longer the target (a sheet or viewer opened on top), **drop the spotlight** rather than ring something underneath.
- When controls are hidden, hide the real mouse pointer (`@media (pointer: fine) { html, html * { cursor: none !important } }`).

### 5.5 Spotlight (§12.6)
One absolutely positioned div animated to the target rect (padding 7 px; 10 px for big targets; radius = the target's radius + padding, ≤30) with
`box-shadow: 0 0 0 2px paper, 0 0 0 7px accent/26%, 0 0 0 200vmax ink/44%`. The 200vmax shadow dims the whole screen except the hole. Spring: stiffness 170, damping 26, mass 0.9. A `requestAnimationFrame` loop re-measures the tracked element every frame (only setting state on >0.5 px change), so the ring follows sheets sliding in, accordions and scroll.

### 5.6 Scroll comfort
Comfort zone = from 76 px (below the sticky top bar) to `innerHeight − max(130 px desktop | 150 px phone, dockHeight + 24)`. If the target is outside it or horizontally clipped, `scrollIntoView({ block: tall ? 'start' : 'center', inline: 'center', behavior: smooth })` with a temporary `scroll-margin-top`, then wait until the rect is still for 4 frames (≤1.4 s).

### 5.7 Subtitle dock placement (§12.8)
The dock sits at the bottom; it moves to the top only if it would cover the spotlit target **and** the top zone is completely clear of it. A target taller than the gap keeps the dock at the bottom (its heading is at its top). Both zones are padded by `SETTLE_PAD = 56` px, and after a big target jump (>24 px) the dock keeps using the previous rect for 750 ms, so it never flips under a spotlight ring that is still springing into place. The dock reports its height (ResizeObserver) to the scroll and cursor logic.

### 5.8 Pause, resume, jump, replay
- **Pause:** abort the run controller, cancel narration, remember `pathname`, reveal the full line.
- **Resume:** restore the seat `seatAt(steps, index)`, navigate back to the remembered page (wait 450–550 ms), then `runFrom(index)` (the step replays its line; a click already done is not repeated, because `actionDone` remembers it).
- **Next:** if the current step is a click not yet done, click it now, then run from i+1 after 250 ms.
- **Replay line:** abort and `runFrom(index)` (no repeated click).
- **Real input only:** listeners on `window` in the capture phase ignore `!e.isTrusted` (the tour's own clicks) and events inside `[data-autopilot-controls]`.

### 5.9 Chapters and cold jumps
`chapter.prepare(actions)` fast-forwards from a **clean reset** using the app's real store actions (not DOM clicks), e.g. "advance this deal, finish intake, build the placement", and may return the route to open. Example: the "Ledger" chapter prepares everything *before* the approval, because the approval is what the chapter shows. Every chapter must play correctly when jumped to cold; the e2e harness checks each one.

### 5.10 Multi-persona tours
`seatAt(steps, i)` = the nearest earlier `seat`. On a change: show the seat card (colour wash in the persona's brand colour, "Now in <Name>'s seat"), wait 380 ms, `setPersona`, navigate to `step.route ?? persona.home`, hold 1220 ms, hide. Persona **before** navigation: a persona's guard won't render another persona's routes. Exiting the tour always returns the app to the primary persona.

### 5.11 Up next
Between chapters (when `tour.upNext`): a card with the next chapter's thumbnail and a 3 s draining countdown; "Go now" resolves early and "Stay here" pauses. The multi-persona tour uses seat cards instead.

---

## 6. Controls and Overlay UI

| Element | Detail |
|---|---|
| **Dock** | Frosted ink glass, max ~720 px wide, 2 lines (3 on a phone), word-by-word reveal (future words at 25% opacity), chapter name + `n / total`, a draining dwell line |
| **Progress rail** | A 2 px bar on the dock edge; chapter ticks; hover or drag shows a thumbnail tooltip; click jumps to that chapter |
| **Buttons** | ◀ ▶ ⏯, chapter grid, voice (Bella / Samantha / Browser voice / Subtitles only / Replay line), seat chip (pauses and switches persona), 1× / 1.25×, ✕ |
| **Tap for sound pill** | Shown when autoplay is blocked; pulses |
| **Chapter overview** | Full-screen grid: thumbnail, title, step count and approximate duration (`estimateDurationMs`), seat badge; tour tabs; keyboard-navigable; owns the keyboard while open |
| **Splash** | Tour tabs + blurb, voice picker, a big Play button |
| **End card** | Two finale lines, Replay, Close |
| **Auto-hide** | Controls hide after 2 s without pointer movement; always shown while paused or while a menu is open |
| **Accessibility** | `prefers-reduced-motion` → no glide, instant spotlight, full lines; Esc always exits; the dock is `aria-live="polite"` |
| **z-index** | Overlay at 2147483000 (above every sheet, popover and toast) |

---

## 7. Narration Pipeline (ElevenLabs)

- **One source of truth:** `NARRATION` exports from the script files. The generator imports them directly.
- **Output:** `public/audio/demo/<voice>/<stepId>.mp3` + `manifest.json` `{ id: { hash, chars, durationMs, text } }`, kept in script order.
- **Incremental:** hash = sha256 of `{ text, voiceId, model, format, settings }`. Only changed lines are regenerated (`--force` and `--only=a,b` to override). Lines that no longer exist are dropped from the manifest.
- **Request stitching:** send `previous_text` / `next_text` (within a tour, not across tours) so intonation flows across separate files.
- **Duration** without decoding: 128 kbps CBR → `bytes × 8 / 128` ms. The app fetches the manifest and uses these durations to pace subtitles when audio is blocked.
- **Settings that worked:** `eleven_multilingual_v2`, `mp3_44100_128`, `{ stability: 0.5, similarity_boost: 0.8, style: 0.25, use_speaker_boost: true }`. Two voices (one bright and warm, one relaxed) let the presenter choose.
- **Key handling:** read `ELEVENLABS_API_KEY` from a git-ignored env file; never print it. Retries 3× on 5xx/429.
- **Cost guide:** ~6,100 characters for 79 lines ≈ 7 min of audio per voice (~6.7 MB per voice at 128 kbps).

---

## 8. Requirements on the Host App

The autopilot is only as reliable as the app's cooperation. Put these in place first:

| Requirement | Why |
|---|---|
| **`data-demo="…"` ids** on every element the tour touches (nav items `nav-<section>`, cards, buttons, tabs, the hero sections) | Stable targets that survive copy changes. `findText` is the fallback, not the plan |
| **Deterministic demo data + `reset()`** | Every run and every chapter starts from the same state |
| **A fixed demo clock** (`now()` = anchor + time since load) | "Today", "overdue" and timestamps read the same at every showing |
| **Store actions for every handoff** (used by `prepare`) | Cold chapter jumps without DOM clicking |
| **Personas in the store** (`setPersona`), each with a `home` route and route guards | Multi-seat tours |
| **Overlays close on `Escape` dispatched on `window`** | `before: closeOverlays` cleans up between chapters |
| **App events for hard-to-click UI** (`window.dispatchEvent(new Event('open-audit'))`) | `fallback` when a control exists only in one tab |
| **Exiting route pages ignore pointer events** | Clicks never land on a page that is about to be destroyed |
| **In-app "AI" animations with visible progress** (file-by-file, step list) | `deliberate` waits never look frozen |
| **Responsive nav with the same hrefs** in the sidebar and the phone tab bar / More menu | `nav()` selectors + `fallbackRoute` work on every size |
| **No `transform` on `html`/`body`** | Fixed-position overlay coordinates match `getBoundingClientRect` |

---

## 9. Observability and Tests

### 9.1 Overlay attributes (read by tests and scripts)
`data-autopilot-state` (idle|splash|running|paused|finished), `data-autopilot-step`, `data-autopilot-index`, `data-autopilot-skipped` (csv), `data-autopilot-gliding`, `data-autopilot-clicks` (count), `data-autopilot-audio` (idle|playing|paused). The dock has `data-demo="dock"` and `data-edge`.

### 9.2 Timing log
`window.__autopilotLog: { id, resolveMs, skipped, lineStart, lineEnd, source }[]`, where `source` is `audio:<ms>` | `speech` | `timer`.

### 9.3 E2E harness (`scripts/autopilot-e2e.mjs`, Playwright, against `vite preview`, not dev)
| Case | Pass criteria |
|---|---|
| `tour` (desktop 1180×820 and phone 390×844, `speed=2&mute=1`) | Reaches `finished`; no console errors; **no skipped steps**; every target resolves in ≤1.5 s (except `deliberate`); **line-to-line gap ≤1.2 s** within chapters (prints max and p95); screenshots of key moments |
| `jump` | Open the overview right after start, jump to a late chapter, reach the end. Repeat for **every chapter** |
| `autoplay` | Chromium with `--autoplay-policy=document-user-activation-required`: running within 2 s, the "Tap for sound" pill shows; one tap starts audio **without pausing**; completes |
| `audio` | Default voice, unmuted at 1× through the first chapter: each MP3 returns 200 and plays with duration > 0 |

Both tours run in all cases (`TOUR=lifecycle`).

### 9.4 Unit tests
`runner.test.ts` (jsdom, visibility injected): selector order and hidden duplicates, role and innermost text lookup, `closest`, never matching the overlay, invalid selectors, `waitForTarget` resolving on mutation and timing out, `requires`, `pickVoice` ranking, duration estimates, **every step has an id and a unique audio file, every chapter has a first step and a thumbnail step**. `dock.test.ts`: the dockEdge cases including the tall-target and settling-ring regressions.

### 9.5 Thumbnails
`capture-thumbs.mjs` runs the tour muted at 2× at 1280×800, hides the overlay, screenshots each chapter's `thumbStep` once it has been on screen for 1.3 s (the finale keeps the end card), and encodes 800×500 WebP with `cwebp -q 82` (~20–30 KB each).

---

## 10. Offline / PWA (optional, recommended for live presentations)

- Precache everything, including `**/*.mp3` (raise `maximumFileSizeToCacheInBytes`).
- **Safari plays `<audio>` only from a 206 response to its Range request**; Workbox's precache route returns 200. Import `sw-range.js` (§12.11) ahead of Workbox to slice cached MP3s.
- **Never reload mid-demo:** `registerType: 'prompt'`, no `skipWaiting`/`clientsClaim`. Apply a waiting update only when the tour isn't running **and** (the page is hidden **or** idle >60 s); otherwise show "Update ready — applies next launch" (§12.10).
- **Before a live showing:** install to the home screen, open it twice (the first launch fills the cache), reset inside the installed app, and test in airplane mode with sound.

---

## 11. Lessons Learned (each one cost a fix round)

1. **Start the line before moving the cursor.** Sequential "move, then talk" leaves ~1.5 s of dead air per step. Measure gaps and fail the build over 1.2 s.
2. **Wait for the outgoing page to unmount** before clicking (§5.2). This was the cause of "the sheet opened and vanished".
3. **Unlock audio synchronously inside the click.** Any `await` before `play()` loses the gesture on iOS.
4. **Autoplay will be blocked.** Never stall: run on subtitles at the recorded pace, offer "Tap for sound", and consume that tap so it neither pauses the tour nor clicks the app.
5. **Only `isTrusted` events pause.** The tour's own synthetic clicks must not pause it.
6. **Make "next" perform the skipped click,** or later steps can't find their state.
7. **Cold chapter jumps via store actions,** never by replaying earlier steps.
8. **Drop the spotlight when a click opens something on top** (`elementFromPoint`).
9. **The dock must never cover the target or its heading,** including during the ring's spring settle (SETTLE_PAD + a 750 ms hold). Write a regression test with the exact rects.
10. **Keep the cursor off the dock** (re-aim within the target).
11. **Nav targets differ by device;** list several selectors, use a short timeout, and fall back to the route.
12. **Hidden duplicates exist** (the phone nav on desktop); check visibility.
13. **Seat before route** in multi-persona tours; exit returns to the primary seat.
14. **Deliberate waits need a reason and visible progress,** and are excluded from gap metrics.
15. **Fixed demo clock and resettable data,** or the narration ("three exceptions tonight") stops matching the screen.
16. **Test on `vite preview`, not dev;** HMR reloads mid-tour.
17. **Safari range requests and PWA update reloads** will break an offline showing unless handled (§10).
18. **Measure real duration** with the harness and quote that, not the estimate.
19. **Step ids are audio filenames;** renaming one silently falls back to browser speech. The unit test checks every step has an audio file.
20. **Strip `?play_demo` from the URL** after starting, so a reload doesn't restart the tour.

---

## 12. Reference Code (tested in the original build)

### 12.1 `types.ts`
```ts
export type DemoAction = 'click' | 'hover' | 'scroll' | 'wait' | 'none'

export interface FindText {
  /** Restrict to an accessible role. `any` matches the smallest element whose text matches. */
  role?: 'button' | 'link' | 'tab' | 'heading' | 'any'
  text: string | RegExp
}

/** Persona kinds in the host app (rename to yours). */
export type SeatKind = string
export type TourId = string

export interface DemoStep {
  id: string
  seat?: SeatKind
  chapter: string
  page?: string
  route?: string
  target?: string | string[]
  findText?: FindText
  closest?: string
  action: DemoAction
  subtitle: string
  dwellMs?: number
  deliberate?: string
  timeoutMs?: number
  audioSrc?: string
  clickDelayMs?: number
  requires?: string
  fallbackRoute?: string
  fallback?: () => void
  before?: () => void
}

export interface DemoChapter {
  id: string
  tourId: TourId
  seat?: SeatKind
  /** Matches DemoStep.chapter. */
  title: string
  /** Step whose key frame is the thumbnail (public/demo/thumbs/<id>.webp). */
  thumbStep: string
  route?: string
  /** Fast-forward from a clean reset to what the first step expects. May return the route to open. */
  prepare?: (actions: StoreActions) => string | void
}

export interface DemoTour {
  id: TourId
  title: string
  blurb: string
  steps: DemoStep[]
  chapters: DemoChapter[]
  /** Show the Up next countdown between chapters. */
  upNext: boolean
  finale: [string, string]
}
```

### 12.2 `runner.ts` (pure, framework-free; copy verbatim)
```ts
import type { DemoStep, FindText } from './types'

export type VisibleFn = (el: Element) => boolean

export function isVisible(el: Element): boolean {
  if (!(el instanceof HTMLElement) && !(el instanceof SVGElement)) return false
  if (el.getClientRects().length === 0) return false
  const style = getComputedStyle(el)
  return style.visibility !== 'hidden' && style.display !== 'none'
}

const ROLE_SELECTORS: Record<NonNullable<FindText['role']>, string> = {
  button: 'button, [role="button"]',
  link: 'a[href], [role="link"]',
  tab: '[role="tab"]',
  heading: 'h1, h2, h3, h4, h5, h6, [role="heading"]',
  any: '*',
}

function accessibleText(el: Element): string {
  const label = el.getAttribute('aria-label')
  const text = (el.textContent ?? '').replace(/\s+/g, ' ').trim()
  return label ? `${label} ${text}` : text
}

function matches(text: string, want: string | RegExp): boolean {
  return typeof want === 'string' ? text.toLowerCase().includes(want.toLowerCase()) : want.test(text)
}

export function findByText(root: ParentNode, find: FindText, visible: VisibleFn = isVisible): Element | null {
  const role = find.role ?? 'any'
  const usable = (el: Element) => !el.closest('[data-autopilot-ui]') && visible(el)
  if (role !== 'any') {
    return Array.from(root.querySelectorAll(ROLE_SELECTORS[role])).find((el) => matches(accessibleText(el), find.text) && usable(el)) ?? null
  }
  const doc = (root as Node).ownerDocument ?? (root as Document)
  const walker = doc.createTreeWalker(root as Node, NodeFilter.SHOW_TEXT)
  for (let node = walker.nextNode(); node; node = walker.nextNode()) {
    const parent = node.parentElement
    if (!parent || !node.textContent?.trim() || /^(SCRIPT|STYLE)$/.test(parent.tagName)) continue
    if (matches(accessibleText(parent), find.text) && usable(parent)) return parent
  }
  return null
}

export function resolveTarget(step: Pick<DemoStep, 'target' | 'findText' | 'closest'>, root: ParentNode = document, visible: VisibleFn = isVisible): Element | null {
  const selectors = step.target == null ? [] : Array.isArray(step.target) ? step.target : [step.target]
  let found: Element | null = null
  for (const sel of selectors) {
    let all: Element[] = []
    try { all = Array.from(root.querySelectorAll(sel)) } catch { continue }
    found = all.find((el) => !el.closest('[data-autopilot-ui]') && visible(el)) ?? null
    if (found) break
  }
  if (!found && step.findText) found = findByText(root, step.findText, visible)
  if (found && step.closest) found = found.closest(step.closest) ?? found
  return found
}

export function hasTarget(step: Pick<DemoStep, 'target' | 'findText'>): boolean {
  return step.target != null || step.findText != null
}

export function waitForTarget(
  step: DemoStep,
  opts: { timeoutMs?: number; signal?: AbortSignal; root?: ParentNode; visible?: VisibleFn; pollMs?: number } = {},
): Promise<Element | null> {
  const { timeoutMs = step.timeoutMs ?? 2500, signal, root = document, visible = isVisible, pollMs = 120 } = opts
  return new Promise((resolve) => {
    const now = resolveTarget(step, root, visible)
    if (now || signal?.aborted) return resolve(now)
    let done = false
    const finish = (el: Element | null) => {
      if (done) return
      done = true
      observer?.disconnect()
      window.clearInterval(poll)
      window.clearTimeout(timer)
      signal?.removeEventListener('abort', onAbort)
      resolve(el)
    }
    const check = () => { const el = resolveTarget(step, root, visible); if (el) finish(el) }
    const onAbort = () => finish(null)
    const observeRoot = (root as Node).nodeType ? (root as Node) : document
    const observer = typeof MutationObserver !== 'undefined' ? new MutationObserver(check) : null
    observer?.observe(observeRoot, { childList: true, subtree: true, attributes: true })
    const poll = window.setInterval(check, pollMs)
    const timer = window.setTimeout(() => {
      console.warn(`[autopilot] step "${step.id}": target not found within ${timeoutMs}ms, skipping`)
      finish(null)
    }, timeoutMs)
    signal?.addEventListener('abort', onAbort)
  })
}

export function shouldSkipForDependency(step: DemoStep, skipped: ReadonlySet<string>): boolean {
  return step.requires != null && skipped.has(step.requires)
}

export const MS_PER_WORD = 380
export const DEFAULT_DWELL_MS = 600
export const DEFAULT_CLICK_DELAY_MS = 650
const CURSOR_MS = 1500
const ROUTE_MS = 480

export function wordCount(text: string): number {
  return text.trim().split(/\s+/).filter(Boolean).length
}
export function speechFallbackMs(text: string, speed = 1): number {
  return (wordCount(text) * MS_PER_WORD) / speed
}
export function estimateDurationMs(steps: DemoStep[], speed = 1): number {
  return steps.reduce((sum, s) => {
    const travel = (hasTarget(s) ? CURSOR_MS : 0) + (s.route ? ROUTE_MS : 0)
    const click = s.action === 'click' ? (s.clickDelayMs ?? DEFAULT_CLICK_DELAY_MS) : 0
    return sum + (travel + click + (s.dwellMs ?? DEFAULT_DWELL_MS)) / speed + speechFallbackMs(s.subtitle, speed)
  }, 0)
}

const PREFERRED_NAMES = ['Samantha', 'Ava', 'Allison', 'Daniel', 'Serena']
/** Premium/Enhanced English voices first, then classic macOS narrators, then any en-US/en-GB, local before network. */
export function pickVoice(voices: SpeechSynthesisVoice[]): SpeechSynthesisVoice | null {
  const english = voices.filter((v) => /^en[-_](US|GB)/i.test(v.lang))
  const pool = english.length ? english : voices.filter((v) => /^en/i.test(v.lang))
  const quality = (v: SpeechSynthesisVoice) => /premium|enhanced|natural|neural/i.test(v.name)
  const rankName = (v: SpeechSynthesisVoice) => { const i = PREFERRED_NAMES.findIndex((n) => v.name.includes(n)); return i === -1 ? PREFERRED_NAMES.length : i }
  const sorted = [...pool].sort((a, b) =>
    Number(quality(b)) - Number(quality(a)) ||
    rankName(a) - rankName(b) ||
    Number(b.localService) - Number(a.localService) ||
    Number(/en[-_]US/i.test(b.lang)) - Number(/en[-_]US/i.test(a.lang)))
  const best = sorted[0]
  if (!best) return null
  if (!quality(best) && rankName(best) === PREFERRED_NAMES.length) return pool.find((v) => v.default) ?? best
  return best
}

/** Voices load asynchronously in Chromium; wait briefly for `voiceschanged`. */
export function loadVoices(timeoutMs = 1500): Promise<SpeechSynthesisVoice[]> {
  if (typeof window === 'undefined' || !('speechSynthesis' in window)) return Promise.resolve([])
  const synth = window.speechSynthesis
  const now = synth.getVoices()
  if (now.length) return Promise.resolve(now)
  return new Promise((resolve) => {
    const done = () => { synth.removeEventListener('voiceschanged', done); window.clearTimeout(t); resolve(synth.getVoices()) }
    const t = window.setTimeout(done, timeoutMs)
    synth.addEventListener('voiceschanged', done)
  })
}

export class Aborted extends Error { constructor() { super('aborted') } }

export function sleep(ms: number, signal?: AbortSignal): Promise<void> {
  return new Promise((resolve, reject) => {
    if (signal?.aborted) return reject(new Aborted())
    const t = window.setTimeout(() => { signal?.removeEventListener('abort', onAbort); resolve() }, ms)
    const onAbort = () => { window.clearTimeout(t); reject(new Aborted()) }
    signal?.addEventListener('abort', onAbort, { once: true })
  })
}
```

### 12.3 `narrator.ts`
```ts
import type { DemoStep } from './types'

export type Narrator = 'voiceA' | 'voiceB' | 'browser' | 'subtitles'
export const NARRATOR_VOICES = ['voiceA', 'voiceB'] as const // keep in sync with VOICES in gen-narration.mjs
export const NARRATORS: { id: Narrator; label: string; hint: string }[] = [
  { id: 'voiceA', label: 'Bella', hint: 'Bright and warm' },
  { id: 'voiceB', label: 'Samantha', hint: 'Relaxed' },
  { id: 'browser', label: 'Browser voice', hint: 'Built into this device' },
  { id: 'subtitles', label: 'Subtitles only', hint: 'No sound' },
]
const KEY = 'app-narrator'
export const DEFAULT_NARRATOR: Narrator = 'voiceA'
export function loadNarrator(): Narrator {
  try { const v = localStorage.getItem(KEY); if (v && NARRATORS.some((n) => n.id === v)) return v as Narrator } catch { /* no storage */ }
  return DEFAULT_NARRATOR
}
export function saveNarrator(n: Narrator) { try { localStorage.setItem(KEY, n) } catch { /* no storage */ } }
export function audioSrcFor(step: DemoStep, narrator: Narrator): string | undefined {
  if (step.audioSrc) return step.audioSrc
  if (!(NARRATOR_VOICES as readonly string[]).includes(narrator)) return undefined
  return `/audio/demo/${narrator}/${step.id}.mp3`
}
```

### 12.4 `start.ts`
```ts
export const START_EVENT = 'autopilot:start'
/** Call from a click handler: audio can only be unlocked inside a user gesture. */
export function startGuidedDemo() { window.dispatchEvent(new Event(START_EVENT)) }
```

### 12.5 Controller core (React; condensed from `Autopilot.tsx`, logic unchanged)
```tsx
// State: status, index, subtitle, revealed, narrator/sessionVoice, needsGesture, speed, spot, cursor (cx, cy motion values),
// pressed, ripple, flash, dwell, upNext, overview/narrator/seat menus, controlsShown, dockH, tourId.
// Refs: tourRef, runRef (AbortController), indexRef, skipped (Set), actionDone (Set), tracked (Element), audioRef,
// voiceRef (SpeechSynthesisVoice), stopNarration, durations (manifest), tourPath, navAt, outgoing, live (all loop inputs).

function unlockMedia() {                       // MUST run synchronously inside the user's click
  const a = audio()
  const first = audioSrcFor(S()[indexRef.current] ?? S()[0], live.current.narrator)
  if (first) {
    a.src = first; a.muted = true
    a.play().then(() => { a.pause(); a.muted = false; a.currentTime = 0 }).catch(() => { a.muted = false })
  }
  if ('speechSynthesis' in window) { try { const u = new SpeechSynthesisUtterance(' '); u.volume = 0; speechSynthesis.speak(u) } catch {} }
}

function narrate(step: DemoStep, signal: AbortSignal): Promise<void> {
  const text = step.subtitle, words = wordCount(text)
  const { muted, narrator, speed: rate, reduce } = live.current
  const src = audioSrcFor(step, narrator)
  let revealTimer = 0, revealing = false
  const startReveal = (totalMs: number) => {
    if (revealing) return; revealing = true; clearInterval(revealTimer)
    if (reduce) return setRevealed(words)
    const per = Math.max(60, (totalMs * 0.82) / words); let n = 0; setRevealed(0)
    const tick = () => { n += 1; setRevealed(n); if (n >= words) clearInterval(revealTimer) }
    tick(); revealTimer = window.setInterval(tick, per)
  }
  return new Promise<void>((resolve) => {
    let settled = false, detach = () => {}
    const done = () => {
      detach(); if (settled) return; settled = true
      clearInterval(revealTimer); clearTimeout(guard); setRevealed(words)
      logStep(step.id, { lineEnd: performance.now() })
      signal.removeEventListener('abort', onAbort); resolve()
    }
    const onAbort = () => { detach(); clearInterval(revealTimer); settled = true; resolve() }
    signal.addEventListener('abort', onAbort, { once: true })
    stopNarration.current = done                      // muting mid-line finishes the line now
    const mark = (source: string) => logStep(step.id, { lineStart: performance.now(), source })
    const fallbackMs = speechFallbackMs(text, rate)
    let guard = window.setTimeout(done, fallbackMs * 2.5 + 4000)
    const timerOnly = () => {
      const recorded = live.current.blocked ? durations.current[step.id] : undefined   // recorded pace when blocked
      const ms = recorded ? recorded / rate : fallbackMs
      mark('timer'); startReveal(ms); clearTimeout(guard); guard = window.setTimeout(done, ms)
    }
    const blocked = () => { live.current.blocked = true; setNeedsGesture(true); timerOnly() }
    const speak = () => {
      if (!('speechSynthesis' in window)) return timerOnly()
      speechSynthesis.cancel()
      const u = new SpeechSynthesisUtterance(text)
      if (voiceRef.current) u.voice = voiceRef.current
      u.lang = voiceRef.current?.lang ?? 'en-US'; u.rate = rate
      let started = false
      u.onstart = () => { started = true; mark('speech'); startReveal(fallbackMs) }
      u.onboundary = (e) => { if (!e.name || e.name === 'word') setRevealed((r) => Math.max(r, wordCount(text.slice(0, e.charIndex + 1)))) }
      u.onend = () => done()
      u.onerror = (e) => (e.error === 'not-allowed' ? (settled ? undefined : blocked()) : done())
      speechSynthesis.speak(u)
      window.setTimeout(() => { if (!started && !settled) { speechSynthesis.cancel(); timerOnly() } }, 1200)
    }
    if (muted || live.current.blocked) return timerOnly()
    if (src) {
      const a = audio()
      const cleanup = (next: () => void) => { detach(); if (!settled) next() }
      const onEnded = () => cleanup(done), onError = () => cleanup(speak)
      const onMeta = () => { if (Number.isFinite(a.duration)) startReveal((a.duration * 1000) / rate) }
      detach = () => { a.removeEventListener('ended', onEnded); a.removeEventListener('error', onError); a.removeEventListener('loadedmetadata', onMeta) }
      a.addEventListener('ended', onEnded); a.addEventListener('error', onError); a.addEventListener('loadedmetadata', onMeta)
      a.muted = false; a.src = src; a.playbackRate = rate
      a.play().then(() => { mark(`audio:${Math.round((a.duration || 0) * 1000)}`); if (a.readyState >= 1) onMeta() })
        .catch((err: unknown) => cleanup((err as Error)?.name === 'NotAllowedError' ? blocked : speak))
      return
    }
    speak()
  })
}

async function playStep(i: number, signal: AbortSignal) {
  const step = S()[i]
  indexRef.current = i; setIndex(i); preload(S()[i + 1])
  if (shouldSkipForDependency(step, skipped.current)) { skipped.current.add(step.id); return }
  step.before?.()

  const seat = seatAt(S(), i); let routed = false
  if (seat !== live.current.seatKind) {                       // persona switch: card → persona → route
    tracked.current = null; setSubtitle(null); setSeatCard(seat)
    try {
      await sleep(380 / live.current.speed, signal)
      actions.setPersona(personaFor(seat)); live.current.seatKind = seat
      await sleep(40, signal)
      navigate(step.route ?? homeFor(seat)); routed = true
      await sleep(1220 / live.current.speed, signal)
    } finally { setSeatCard(null) }
    await sleep(250, signal)
  }
  if (!routed && step.route && live.current.pathname !== step.route) { tracked.current = null; navigate(step.route); await sleep(480, signal) }

  if (step.page) {                                             // §5.2 page settling
    const onPage = () => (step.page === '/' ? live.current.pathname === '/' : live.current.pathname.startsWith(step.page!))
    const t0 = performance.now()
    while (!onPage() && performance.now() - t0 < 2500) await sleep(50, signal)
    const settle = 320 - (performance.now() - navAt.current); if (settle > 0) await sleep(settle, signal)
    const t1 = performance.now()
    const swapped = () => { const cur = document.querySelector('main')?.firstElementChild ?? null; return !outgoing.current || !outgoing.current.isConnected || cur !== outgoing.current }
    let nudged = false
    while (!swapped() && performance.now() - t1 < 2500) {
      await sleep(40, signal)
      if (!nudged && performance.now() - t1 > 400) { nudged = true; navigate(location.pathname + location.search, { replace: true, state: { autopilotFlush: Date.now() } }) }
    }
    if (performance.now() - t1 > 60) await sleep(200, signal)
  }

  let el: Element | null = null
  if (hasTarget(step)) {
    const t0 = performance.now()
    el = await waitForTarget(step, { signal })
    if (signal.aborted) throw new Aborted()
    logStep(step.id, { resolveMs: Math.round(performance.now() - t0), skipped: !el && !step.fallbackRoute && !step.fallback })
    if (!el) {
      if (step.fallbackRoute) { tracked.current = null; navigate(step.fallbackRoute); actionDone.current.add(step.id) }
      else if (step.fallback) { step.fallback(); actionDone.current.add(step.id) }
      else { skipped.current.add(step.id); return }
    }
  }

  setSubtitle(step.subtitle); setRevealed(0)
  const narration = narrate(step, signal)                      // line starts NOW, under the movement

  if (el) {
    tracked.current = step.action === 'wait' ? null : el
    await scrollIntoComfort(el, signal)
    if (step.action !== 'wait' && step.action !== 'none') { setCursorOn(true); const p = pointFor(el); await glideTo(p.x, p.y, signal); setIdleOver(true) }
    if (step.action === 'click' && !actionDone.current.has(step.id)) {
      await sleep((step.clickDelayMs ?? DEFAULT_CLICK_DELAY_MS) / live.current.speed, signal)
      const target = el.isConnected ? el : resolveTarget(step)
      if (target) {
        const p = pointFor(target)
        if (Math.hypot(p.x - cx.get(), p.y - cy.get()) > 6) await glideTo(p.x, p.y, signal)
        actionDone.current.add(step.id)
        pressAndClick(target)
        await sleep(500, signal)
        const b = target.getBoundingClientRect()
        const top = target.isConnected ? document.elementFromPoint(b.left + b.width / 2, b.top + b.height / 2) : null
        if (!top || !(target === top || target.contains(top))) tracked.current = null   // something opened on top
      }
    }
  } else tracked.current = null

  await narration
  if (signal.aborted) throw new Aborted()
  const hold = (step.dwellMs ?? DEFAULT_DWELL_MS) / live.current.speed
  setDwell({ key: i, ms: hold })
  try { await sleep(hold, signal) } finally { setDwell(null) }
}

const runFrom = async (start: number) => {
  runRef.current?.abort(); const ctrl = new AbortController(); runRef.current = ctrl; setStatus('running')
  try {
    for (let i = start; i < S().length; i++) {
      await playStep(i, ctrl.signal)
      const next = S()[i + 1]
      if (tourRef.current.upNext && next && next.chapter !== S()[i].chapter && next.chapter !== 'Finale') await upNextPause(i + 1, ctrl.signal)
    }
    if (!ctrl.signal.aborted) { tracked.current = null; setSpot(null); setCursorOn(false); setStatus('finished') }
  } catch (err) { if (!(err instanceof Aborted)) { console.warn('[autopilot] step failed, pausing', err); setStatus('paused') } }
}

// Track route changes for page settling:
useEffect(() => {
  navAt.current = performance.now()
  outgoing.current = firstPath.current ? null : (document.querySelector('main')?.firstElementChild ?? null)
  firstPath.current = false
}, [location.pathname])

// ?play_demo=true: strip params, then begin({ gesture: false }). ?demo=1: show the splash.
// Blocked autoplay: on the first trusted pointerdown/keydown (capture phase), preventDefault + stopImmediatePropagation,
// also swallow the following pointerup/mouseup/click for 700 ms, set blocked=false, unlockMedia(), abort, runFrom(index).
// Pause on trusted input outside [data-autopilot-controls]; Space/k toggles; ←/→ jump; Esc exits (closes a menu first).
```

### 12.6 `Spotlight.tsx`
```tsx
export function Spotlight({ rect, flash, reduce }: { rect: Rect | null; flash: number; reduce: boolean }) {
  const spring = reduce ? { duration: 0 } : { type: 'spring' as const, stiffness: 170, damping: 26, mass: 0.9 }
  const [lastRect, setLastRect] = useState<Rect | null>(rect)
  if (rect && rect !== lastRect) setLastRect(rect)
  const r = rect ?? lastRect                                   // keep the last rect while fading out
  return (
    <motion.div aria-hidden className="pointer-events-none fixed inset-0" style={{ zIndex: Z - 3 }}
      initial={false} animate={{ opacity: rect ? 1 : 0 }} transition={{ duration: reduce ? 0 : 0.35 }}>
      {r ? (
        <motion.div className="absolute top-0 left-0" initial={false}
          animate={{ x: r.x, y: r.y, width: r.w, height: r.h, borderRadius: r.r }} transition={spring}
          style={{ boxShadow: `0 0 0 2px rgba(243,239,230,.95), 0 0 0 7px rgba(168,67,45,.26), 0 0 0 200vmax rgba(21,33,59,.44)` }}>
          <AnimatePresence>
            {flash ? (
              <motion.span key={flash} className="absolute inset-0" style={{ borderRadius: 'inherit', boxShadow: `0 0 0 3px ${ACCENT}, 0 0 24px 6px rgba(168,67,45,.45)` }}
                initial={{ opacity: 0.95 }} animate={{ opacity: 0 }} transition={{ duration: reduce ? 0 : 0.9 }} />
            ) : null}
          </AnimatePresence>
        </motion.div>
      ) : null}
    </motion.div>
  )
}

// Tracking loop (runs while running/paused):
useEffect(() => {
  if (status !== 'running' && status !== 'paused') return
  let raf = 0, last: Rect | null = null
  const loop = () => {
    const el = tracked.current; let next: Rect | null = null
    if (el?.isConnected) {
      const b = el.getBoundingClientRect()
      if (b.width > 0 && b.height > 0) {
        const pad = b.width > 520 || b.height > 320 ? 10 : 7
        const radius = parseFloat(getComputedStyle(el).borderTopLeftRadius) || 8
        next = { x: b.left - pad, y: b.top - pad, w: b.width + pad * 2, h: b.height + pad * 2, r: Math.min(30, radius + pad) }
      }
    }
    const moved = (last == null) !== (next == null) || (last && next && (Math.abs(last.x - next.x) > 0.5 || Math.abs(last.y - next.y) > 0.5 || Math.abs(last.w - next.w) > 0.5 || Math.abs(last.h - next.h) > 0.5))
    if (moved) { last = next; setSpot(next) }
    raf = requestAnimationFrame(loop)
  }
  raf = requestAnimationFrame(loop)
  return () => cancelAnimationFrame(raf)
}, [status])
```

### 12.7 Cursor motion, scroll and click
```ts
function pointFor(el: Element) {
  const b = el.getBoundingClientRect(), vw = innerWidth, vh = innerHeight
  const big = b.width > 280 || b.height > 120
  let x = big ? b.left + Math.min(b.width * 0.62, b.width - 24) : b.left + b.width / 2
  let y = big ? b.top + Math.min(b.height * 0.4, 110) : b.top + b.height / 2
  x = Math.max(12, Math.min(vw - 24, x)); y = Math.max(72, Math.min(vh - 24, y))
  const dock = document.querySelector('[data-demo="dock"]')?.getBoundingClientRect()
  if (dock && dock.height > 0 && y > dock.top - 24 && y < dock.bottom + 24 && x > dock.left && x < dock.right) {
    const above = Math.max(b.top + 12, 72), below = Math.min(b.bottom - 12, vh - 24)
    if (dock.top - 28 >= above) y = dock.top - 28
    else if (dock.bottom + 28 <= below) y = dock.bottom + 28
  }
  return { x, y }
}

async function glideTo(x: number, y: number, signal: AbortSignal) {
  const x0 = cx.get(), y0 = cy.get()
  if (live.current.reduce || x0 < 0) { cx.set(x); cy.set(y); return }
  const dx = x - x0, dy = y - y0, dist = Math.hypot(dx, dy)
  if (dist < 2) return
  const bow = Math.min(140, dist * 0.22)
  const mx = (x0 + x) / 2 - (dy / dist) * bow, my = (y0 + y) / 2 + (dx / dist) * bow   // control point, perpendicular
  const duration = Math.min(0.9, Math.max(0.6, 0.45 + dist / 1800)) / live.current.speed
  setIdleOver(false); setGliding(true)
  await new Promise<void>((resolve) => {
    const c = animate(0, 1, { type: 'spring', duration, bounce: 0.16, onComplete: resolve, onUpdate: (t) => {
      const u = 1 - t
      cx.set(u * u * x0 + 2 * u * t * mx + t * t * x); cy.set(u * u * y0 + 2 * u * t * my + t * t * y)
    } })
    signal.addEventListener('abort', () => { c.stop(); resolve() }, { once: true })
  })
  setGliding(false)
  if (signal.aborted) throw new Aborted()
}

async function scrollIntoComfort(el: Element, signal: AbortSignal) {
  const b = el.getBoundingClientRect(), vh = innerHeight, top = 76
  const bottom = vh - Math.max(phone ? 150 : 130, live.current.dockH + 24)
  if (b.top >= top && b.bottom <= bottom && b.left >= 0 && b.right <= innerWidth) return
  const tall = b.height > (bottom - top) * 0.8
  const html = el as HTMLElement, prev = html.style.scrollMarginTop
  html.style.scrollMarginTop = `${top + 12}px`
  el.scrollIntoView({ behavior: live.current.reduce ? 'auto' : 'smooth', block: tall ? 'start' : 'center', inline: 'center' })
  html.style.scrollMarginTop = prev
  let last = el.getBoundingClientRect(), still = 0; const t0 = performance.now()
  while (still < 4 && performance.now() - t0 < 1400) {
    await sleep(40, signal)
    const now = el.getBoundingClientRect()
    still = Math.abs(now.top - last.top) < 0.5 && Math.abs(now.left - last.left) < 0.5 ? still + 1 : 0
    last = now
  }
}

function pressAndClick(el: Element) {
  const b = el.getBoundingClientRect()
  const init = { bubbles: true, cancelable: true, clientX: b.left + b.width / 2, clientY: b.top + b.height / 2, pointerId: 1, isPrimary: true, button: 0 }
  setPressed(true); setRipple((r) => r + 1); setFlash((f) => f + 1)
  try { el.dispatchEvent(new PointerEvent('pointerdown', init)) } catch {}
  window.setTimeout(() => { try { el.dispatchEvent(new PointerEvent('pointerup', init)) } catch {}; (el as HTMLElement).click(); setPressed(false) }, 110)
}
```

### 12.8 `dockEdge` (+ tests)
```ts
const SETTLE_PAD = 56
export function dockEdge(avoid: Rect | null, vh: number, dockH: number, bottomGap: number, topGap: number): 'top' | 'bottom' {
  if (!avoid) return 'bottom'
  const overlap = (a0: number, a1: number) => Math.max(0, Math.min(a1, avoid.y + avoid.h) - Math.max(a0, avoid.y))
  const bottom = overlap(vh - bottomGap - dockH - SETTLE_PAD, vh - bottomGap + SETTLE_PAD)
  if (bottom === 0) return 'bottom'
  const top = overlap(topGap - SETTLE_PAD, topGap + dockH + SETTLE_PAD)
  return top === 0 ? 'top' : 'bottom'
}
// In SubtitleDock: bottomGap = phone ? 76 : 20 (above a phone tab bar), topGap = phone ? 68 : 76.
// Hold the previous avoid rect for 750 ms when it jumps by > 24 px before passing it to dockEdge.
```
```ts
// dock.test.ts
const vh = 820, dockH = 69, bottomGap = 20, topGap = 76
expect(dockEdge({ x: 0, y: 100, w: 300, h: 200, r: 14 }, vh, dockH, bottomGap, topGap)).toBe('bottom')   // clear
expect(dockEdge({ x: 0, y: 500, w: 300, h: 300, r: 14 }, vh, dockH, bottomGap, topGap)).toBe('top')      // low target
expect(dockEdge({ x: 769, y: 78, w: 373, h: 797, r: 14 }, vh, dockH, bottomGap, topGap)).toBe('bottom')  // taller than gap
expect(dockEdge({ x: 15, y: 760.4, w: 318, h: 61.98, r: 14 }, 844, 84.16, 76, 68)).toBe('top')           // ring still settling
```

### 12.9 `scripts/gen-narration.mjs`
```js
#!/usr/bin/env node
// node scripts/gen-narration.mjs --voice voiceA | --voice all [--force] [--only=intro,finale]
import { createHash } from 'node:crypto'
import { existsSync, mkdirSync, readFileSync, writeFileSync } from 'node:fs'
import { dirname, join, resolve } from 'node:path'
import { fileURLToPath } from 'node:url'

const APP = resolve(dirname(fileURLToPath(import.meta.url)), '..')
const AUDIO = join(APP, 'public/audio/demo')
const VOICES = { voiceA: '<elevenlabs-voice-id>', voiceB: '<elevenlabs-voice-id>' } // keep in sync with narrator.ts
const MODEL_ID = 'eleven_multilingual_v2', OUTPUT_FORMAT = 'mp3_44100_128'
const VOICE_SETTINGS = { stability: 0.5, similarity_boost: 0.8, style: 0.25, use_speaker_boost: true }
const TOURS = [['src/demo/script.ts', 'NARRATION'] /*, ['src/demo/lifecycle.ts', 'LIFECYCLE_NARRATION'] */]

const args = process.argv.slice(2)
const force = args.includes('--force')
const only = args.find((a) => a.startsWith('--only='))?.slice(7).split(',').filter(Boolean)
const voiceArg = args.find((a) => a.startsWith('--voice='))?.slice(8) ?? args[args.indexOf('--voice') + 1]
const voiceNames = !voiceArg || voiceArg.startsWith('--') ? [Object.keys(VOICES)[0]] : voiceArg === 'all' ? Object.keys(VOICES) : [voiceArg]

function readKey() {
  if (process.env.ELEVENLABS_API_KEY) return process.env.ELEVENLABS_API_KEY
  for (const f of [resolve(APP, '.dev.vars'), resolve(APP, '..', '.dev.vars'), resolve(APP, '.env')]) {
    if (!existsSync(f)) continue
    const line = readFileSync(f, 'utf8').split('\n').find((l) => /^\s*ELEVENLABS_API_KEY\s*=/.test(l))
    const key = line?.split('=').slice(1).join('=').trim().replace(/^["']|["']$/g, '')
    if (key) return key
  }
  throw new Error('ELEVENLABS_API_KEY not set (env, .dev.vars or .env)')
}
const hashOf = (text, voiceId) => createHash('sha256').update(JSON.stringify({ text, voiceId, MODEL_ID, OUTPUT_FORMAT, VOICE_SETTINGS })).digest('hex').slice(0, 16)

async function synth(key, voiceId, text, previous, next) {
  const url = `https://api.elevenlabs.io/v1/text-to-speech/${voiceId}?output_format=${OUTPUT_FORMAT}`
  for (let attempt = 1; attempt <= 3; attempt++) {
    const res = await fetch(url, {
      method: 'POST',
      headers: { 'xi-api-key': key, 'content-type': 'application/json', accept: 'audio/mpeg' },
      body: JSON.stringify({ text, model_id: MODEL_ID, voice_settings: VOICE_SETTINGS, ...(previous ? { previous_text: previous } : {}), ...(next ? { next_text: next } : {}) }),
    })
    if (res.ok) return Buffer.from(await res.arrayBuffer())
    const detail = (await res.text()).slice(0, 300)
    if (attempt === 3 || (res.status < 500 && res.status !== 429)) throw new Error(`ElevenLabs ${res.status}: ${detail}`)
    await new Promise((r) => setTimeout(r, 1500 * attempt))
  }
}

async function generate(key, tours, voice) {
  const voiceId = VOICES[voice], OUT = join(AUDIO, voice), MANIFEST = join(OUT, 'manifest.json')
  mkdirSync(OUT, { recursive: true })
  const manifest = existsSync(MANIFEST) ? JSON.parse(readFileSync(MANIFEST, 'utf8')) : {}
  let generated = 0, chars = 0
  for (const lines of tours) {
    for (let i = 0; i < lines.length; i++) {
      const { id, subtitle } = lines[i], file = join(OUT, `${id}.mp3`), hash = hashOf(subtitle, voiceId)
      if ((only && !only.includes(id)) || (!force && existsSync(file) && manifest[id]?.hash === hash)) continue
      const audio = await synth(key, voiceId, subtitle, lines[i - 1]?.subtitle, lines[i + 1]?.subtitle)  // stitching within a tour
      writeFileSync(file, audio)
      const durationMs = Math.round((audio.length * 8) / 128) // 128 kbps CBR
      manifest[id] = { hash, chars: subtitle.length, durationMs, text: subtitle }
      chars += subtitle.length; generated++
      console.log(`  ${voice} ${id.padEnd(18)} ${(durationMs / 1000).toFixed(1)}s`)
    }
  }
  const ordered = {}
  for (const lines of tours) for (const { id } of lines) if (manifest[id]) ordered[id] = manifest[id]
  writeFileSync(MANIFEST, JSON.stringify(ordered, null, 2) + '\n')
  console.log(`${voice}: generated ${generated}, ${chars} chars, total ${(Object.values(ordered).reduce((s, m) => s + m.durationMs, 0) / 1000).toFixed(0)} s`)
}

const tours = []
for (const [file, name] of TOURS) tours.push((await import(join(APP, file)))[name])   // Node ≥ 22.6 imports .ts
const key = readKey()
for (const v of voiceNames) await generate(key, tours, v)
```

### 12.10 PWA update policy (pure, unit-testable)
```ts
export const IDLE_MS = 60_000
export function canApplyUpdate(env: { isTourRunning(): boolean; isHidden(): boolean; now(): number; lastInputAt(): number }) {
  if (env.isTourRunning()) return false
  return env.isHidden() || env.now() - env.lastInputAt() > IDLE_MS
}
// isTourRunning = () => document.querySelector('[data-autopilot-ui]')?.getAttribute('data-autopilot-state') === 'running'
// vite-plugin-pwa: registerType 'prompt'; onNeedRefresh → pending; check on a timer and on visibilitychange → updateSW(true).
```

### 12.11 `public/sw-range.js` (Safari audio from the precache)
```js
self.addEventListener('fetch', (event) => {
  const req = event.request, range = req.headers.get('range')
  if (req.method !== 'GET' || !range) return
  const url = new URL(req.url)
  if (url.origin !== self.location.origin || !url.pathname.endsWith('.mp3')) return
  event.stopImmediatePropagation()
  event.respondWith((async () => {
    const cached = await caches.match(url.pathname, { ignoreSearch: true })
    if (!cached) return fetch(req)
    const blob = await cached.blob(), size = blob.size
    const m = /^bytes=(\d*)-(\d*)$/.exec(range.trim())
    let start = m && m[1] ? Number(m[1]) : 0, end = m && m[2] ? Number(m[2]) : size - 1
    if (m && !m[1] && m[2]) { start = Math.max(0, size - Number(m[2])); end = size - 1 }
    end = Math.min(end, size - 1)
    if (!m || start > end) return new Response(null, { status: 416, headers: { 'Content-Range': `bytes */${size}` } })
    return new Response(blob.slice(start, end + 1), { status: 206, statusText: 'Partial Content', headers: {
      'Content-Type': cached.headers.get('Content-Type') || 'audio/mpeg', 'Content-Length': String(end - start + 1),
      'Content-Range': `bytes ${start}-${end}/${size}`, 'Accept-Ranges': 'bytes' } })
  })())
})
// workbox: { importScripts: ['/sw-range.js'], globPatterns: ['**/*.{js,css,html,svg,png,webp,woff2,json,mp3}'], maximumFileSizeToCacheInBytes: 6 * 1024 * 1024 }
```

---

## 13. Dependencies

| Package | Purpose |
|---|---|
| `motion` (Framer Motion v11+; `motion/react`) | Springs for the cursor, spotlight, dock, cards; `useMotionValue`, `animate`, `AnimatePresence`, `useReducedMotion` |
| `react-router` (v6.4+/v7) | `useNavigate`, `useLocation` |
| `lucide-react` | Control icons |
| `@playwright/test` | E2E harness, thumbnail capture |
| `vitest` + `jsdom` | Unit tests |
| `vite-plugin-pwa` (optional) | Offline precache |
| `cwebp` (CLI) | Thumbnail encoding |
| ElevenLabs API | Narration (`ELEVENLABS_API_KEY`) |

Browser APIs: `MutationObserver`, `requestAnimationFrame`, `HTMLAudioElement`, `speechSynthesis`, `PointerEvent`, `elementFromPoint`, `ResizeObserver`, `matchMedia`.

Other frameworks: `runner.ts`, `narrator.ts`, `dockEdge`, the narration script and the harness are framework-free. Port the controller (§12.5) as a store or class; the only React-specific parts are state setters and motion values.

---

## 14. The Prompt Kit (drop-in)

Use these with Claude Code (or any coding agent) in the target repo. **Copy this file into the repo first** (e.g. `docs/features/guided-demo-autopilot.md`) so every prompt can reference it. Run them in order; each ends at a verifiable checkpoint. Prompt 0 alone does the whole build when the app is small; for a large app, run 1–8 one at a time.

### Prompt 0 · Whole build in one go
```
Read docs/features/guided-demo-autopilot.md end to end. Build that feature in this app exactly as specified:
the self-driving narrated tour with cursor, spotlight, subtitle dock, controls, chapters, deep links,
autoplay-safe narration (pre-recorded MP3 → speech → timer), unit tests and the Playwright e2e harness.

Context you must gather first (don't ask me unless truly ambiguous):
- the app's framework, router, state store, styling system and brand colours (match them in the overlay);
- the main user journey a buyer should see in ~4–5 minutes (read README, docs/, notes/, any demo or sales
  script; if none exists, propose one: 8–10 chapters, 35–50 steps, ending on the strongest outcome);
- the vocabulary customers use (transcripts, notes); use their words in the lines.

Then work in this order, committing after each stage and running the checks named in the spec:
1. Host-app readiness (§8): data-demo ids, deterministic reset(), fixed demo clock, store actions for
   chapter prepare(), Escape closes overlays, exiting pages ignore taps.
2. src/demo/types.ts, runner.ts, narrator.ts, start.ts copied from §12 (adapt names only) + runner.test.ts.
3. The script(s) (§4) with NARRATION and CHAPTER_META exports. Follow §4.3 for every line.
4. Autopilot.tsx controller (§5, §12.5) + overlay (Spotlight, Cursor, SubtitleDock with dockEdge, Splash,
   EndCard, ChapterOverview, UpNextCard, ProgressRail, NarratorMenu) + dock.test.ts. Mount it once in the
   app shell; add "Play guided demo" to the user menu (calls startGuidedDemo() inside the click).
5. scripts/gen-narration.mjs (§12.9). If ELEVENLABS_API_KEY is available, generate one voice; otherwise
   leave the browser-voice fallback working and tell me the command to run.
6. scripts/autopilot-e2e.mjs (§9.3) against `vite preview`; iterate on the script/app until it passes on
   desktop 1180×820 and phone 390×844: no skipped steps, targets ≤1.5 s, line gaps ≤1.2 s, every chapter
   jump completes, autoplay case passes.
7. scripts/capture-thumbs.mjs (§9.5) for the chapter thumbnails.
8. (If the app is a PWA) §10: precache mp3, sw-range.js, never reload mid-tour.

Respect every item in §11 (Lessons learned). Report the measured tour length, the step count, any steps
you had to skip or mark deliberate (with reasons), and the e2e summary.
```

### Prompt 1 · Readiness audit and instrumentation
```
Read docs/features/guided-demo-autopilot.md §8 and §11. Audit this app for autopilot readiness and fix the gaps:
- add data-demo="..." to every element the demo journey touches (list them as a table: page, element,
  id, why); nav items as data-demo="nav-<section>" in BOTH the desktop nav and the phone nav/More menu;
- a deterministic seed + store reset(); a fixed demo clock (anchor + time since load) used everywhere a
  date or relative time is displayed (add a test that greps for new Date()/Date.now() outside the clock);
- store actions for every state change the tour performs, so chapters can fast-forward without DOM clicks;
- overlays/sheets close on a window Escape keydown; exiting route pages ignore pointer events;
- any in-app "AI is working" animation shows visible step-by-step progress for its whole duration.
Don't build the tour yet. Run the existing tests and show me the table.
```

### Prompt 2 · Write the script (the talk track)
```
Read docs/features/guided-demo-autopilot.md §4 (especially §4.3). Write src/demo/script.ts: the guided tour
for <AUDIENCE, e.g. "the CFO evaluating us">, about 4–5 minutes at 1×, 8–10 chapters, 35–50 steps.

Inputs: <paste the demo script / journey / transcript excerpts, or point to files>.
Rules: one idea per step, ≤22 words, spoken English, the customer's own phrases verbatim where possible,
outcomes not UI instructions, concrete on-screen numbers, name the human approval wherever AI acts,
the line must be true *while* its click happens. Every target uses data-demo ids from Prompt 1 with a
findText fallback; nav steps use nav() + timeoutMs 700 + fallbackRoute; long in-app animations get
deliberate + timeoutMs 9000 + dwellMs 1000; the finale has dwellMs 3500.
Export DEMO_SCRIPT, NARRATION and CHAPTER_META (type-only imports so Node can import the file).
Then print the script as a table (chapter, id, action, target, words, line) and the estimateDurationMs total.
```

### Prompt 3 · Engine and overlay
```
Implement the controller and overlay from docs/features/guided-demo-autopilot.md §5, §6 and §12.5–12.8,
matching this app's design system (fonts, colours, radii) for the dock, cursor, cards and splash.
Requirements: the async step loop with AbortController and a `live` ref; page settling (§5.2) using the
router's location; narration fallback chain with word reveal and a blocked-autoplay path; curved spring
cursor; giant-box-shadow spotlight tracked every frame; scroll comfort zone; dockEdge with SETTLE_PAD and
the 750 ms hold; pause on trusted input only; Space/k, ←/→, Esc; chapter overview with cold jumps via
prepare(); up-next countdown; 1×/1.25×; voice menu persisted; deep links (?play_demo=true&tour=&chapter=
&speed=&voice=&mute=1, ?demo=1 splash; strip params after start); data-autopilot-* attributes and
window.__autopilotLog. Mount once in the app shell; add "Play guided demo" to the user menu.
Add runner.test.ts and dock.test.ts from §9.4. Run the unit tests and play the tour once in a browser.
```

### Prompt 4 · Narration audio
```
Create scripts/gen-narration.mjs from docs/features/guided-demo-autopilot.md §7 and §12.9.
Voices: <voiceA name + ElevenLabs voice id>, <voiceB …>. Key: ELEVENLABS_API_KEY from <.dev.vars / env>;
never print it. Generate --voice all. Then verify: every NARRATION id has an mp3 and a manifest entry for
each voice (add this as a unit test), report total characters and audio minutes, and play the first
chapter in a browser with sound to confirm the subtitles track the voice.
```

### Prompt 5 · Chapters, thumbnails and a second tour
```
Using docs/features/guided-demo-autopilot.md §5.9–5.11 and §9.5:
- write chapters.ts with a prepare() per chapter that fast-forwards from a clean reset using store actions
  (the chapter's own key action must still be pending, because that's what it shows);
- add capture-thumbs.mjs and generate public/demo/thumbs/<chapter>.webp;
- (optional) add a second tour "<NAME>" that follows one record across personas <LIST>, with a seat title
  card per switch (persona BEFORE navigation), its own NARRATION and CHAPTER_META, and generate its audio.
Verify every chapter of every tour plays correctly when jumped to cold.
```

### Prompt 6 · E2E harness and the tuning loop
```
Create scripts/autopilot-e2e.mjs per docs/features/guided-demo-autopilot.md §9.3 (Playwright, against
`npm run build && vite preview`, never the dev server). Cases: tour (desktop 1180×820, phone 390×844,
speed 2, muted), jump (every chapter), autoplay (--autoplay-policy=document-user-activation-required:
pill shows, one tap starts audio without pausing), audio (default voice unmuted, files 200, duration>0),
for each tour. Print gaps (max, p95, anything >1.2 s within a chapter fails), slow targets (>1.5 s,
except deliberate), skipped steps, console errors; save screenshots and e2e-summary.txt.
Then loop: run it, fix the root cause of every failure in the app or the script (not by loosening
thresholds), rerun, until all PASS on both sizes. Report the final summary and measured durations.
```

### Prompt 7 · Offline / live-presentation hardening (PWA apps)
```
Apply docs/features/guided-demo-autopilot.md §10 and §12.10–12.11: precache all assets including the mp3s,
import sw-range.js ahead of Workbox, registerType 'prompt' with the pure canApplyUpdate policy (unit-tested)
so an update never reloads during a running tour. Write a check script that loads once, goes offline,
reloads and plays the tour with audio; and one that publishes an update mid-tour and asserts no reload.
Add a "before you present" checklist to the README (install to the home screen, open twice, reset inside
the app, airplane-mode audio test).
```

### Prompt 8 · Polish review (run after any UI change)
```
Play both tours at 1× with sound on an iPhone-sized (390×844), an iPad-sized (1024×1366) and a laptop
(1440×900) viewport. For every step, check against docs/features/guided-demo-autopilot.md §11: the dock
never covers the target or its heading (including while the ring settles), the cursor never sits on the
dock, nothing is clicked on a page that is leaving, the spotlight drops when a sheet opens on top, no
sideways scrolling on the phone, no dead air >1.2 s, the subtitle matches what's on screen (numbers,
names, dates). List each issue with step id, viewport and a screenshot; fix them; rerun the e2e.
```

### Prompt 9 · Keep it in sync (whenever a screen or a line changes)
```
I changed <screens / lines>. Update the demo: fix any data-demo ids or targets the change broke, edit the
affected subtitles, regenerate only the changed narration (gen-narration.mjs, hash-based), recapture the
thumbnails for affected chapters, update chapter prepare() if the state flow changed, and rerun the e2e
for both tours. Don't rename step ids unless the step's meaning changed (ids are audio filenames).
```

---

## 15. Porting Checklist

- [ ] **Host readiness:** `data-demo` ids, deterministic `reset()`, fixed demo clock, store actions for handoffs, Escape closes overlays, exiting pages ignore taps, visible progress during in-app animations (§8)
- [ ] **Core:** `types.ts`, `runner.ts`, `narrator.ts`, `start.ts` (§12.1–12.4) + `runner.test.ts`
- [ ] **Script:** steps with ids/chapters/targets/lines; `NARRATION` + `CHAPTER_META`; lines follow §4.3
- [ ] **Controller:** async loop + AbortController + `live` ref; seat → route → page settle → target → **narrate first** → scroll → glide → click → await line → dwell (§2.2, §12.5)
- [ ] **Narration chain:** MP3 → speech → timer; synchronous unlock in the click; blocked-autoplay path with "Tap for sound" and gesture consumption; word reveal; preload next
- [ ] **Overlay:** Spotlight (box-shadow hole, rAF tracking, drop when covered), Cursor (Bézier spring, halo, ripple), SubtitleDock (dockEdge + SETTLE_PAD + hold), progress rail, controls with auto-hide, Splash, EndCard, ChapterOverview, UpNext, SeatCard
- [ ] **Input:** pause on trusted input only; Space/k, ←/→, Esc; Resume restores seat and page; Next performs the pending click
- [ ] **Chapters:** `prepare()` cold jumps; thumbnails; up-next countdown; multi-persona seat cards (persona before route); exit to the primary seat
- [ ] **Deep links:** `?play_demo=true&tour&chapter&speed&voice&mute`, `?demo=1`; strip params after start
- [ ] **Audio:** `gen-narration.mjs` (hash manifest, stitching, durations); a test that every step has audio for every voice
- [ ] **Observability:** `data-autopilot-*` attributes, `window.__autopilotLog`
- [ ] **E2E:** tour / jump-every-chapter / autoplay / audio on desktop + phone, on `vite preview`; gaps ≤1.2 s, targets ≤1.5 s, no skips
- [ ] **(PWA)** precache mp3, `sw-range.js`, no reload mid-tour, offline check
- [ ] **Accessibility:** reduced motion, Esc, `aria-live` dock, 44 px controls

### Acceptance criteria (copy into the PR)
1. "Play guided demo" from the user menu starts with sound on iOS Safari, Android Chrome and desktop Chrome/Safari/Arc.
2. `/?play_demo=true` starts within 2 s with no tap; if sound is blocked, the pill shows and one tap enables it without pausing or clicking the app.
3. Both tours finish on phone and desktop with **0 skipped steps, 0 console errors, max line gap ≤1.2 s** (excluding deliberate and chapter transitions).
4. Every chapter plays correctly when jumped to cold.
5. Any tap pauses; Resume continues on the same step, seat and page; Esc exits and returns the app to the primary persona.
6. The subtitle dock never covers the spotlit target or its heading; the cursor never sits on the dock.
7. (PWA) Works in airplane mode after two launches; an app update never reloads during a running tour.

---

<!-- Part of the ShipFactory feature spec library — https://github.com/vishalquantana/shipfactory -->

## About Us

We are [Quantana](https://quantana.com.au), an AI-first design and development agency working with Fortune 500s to build bespoke AI solutions and provide the audit and training needed to ensure success. [Click here to learn more](https://quantana.com.au).

## License

MIT
