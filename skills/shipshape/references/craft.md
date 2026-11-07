# Craft: the visual system and motion

Numbers here are defaults. A project's own tokens always win.

## Layout and hierarchy

- Write a word budget before building a hero or illustration: count the real strings the screen needs (usually 3-7). Everything else is quiet or placeholder.
- Cut before you refine: use as few panels and words as the idea needs, and make every border or badge justify itself.
- Text sits 12px inside any box edge (never under 8px). Neighbouring objects get a fixed 8-12px gap. Edges either align exactly or sit at least 8px apart; 1-3px near-misses look like mistakes.
- A title sits closer to its own metadata than to the next item. Rows get 12px vertical padding (8px when dense).
- Columns line up: rows share one grid template with fixed tracks. A control hugs its content; do not stretch it with `flex: 1` and push its contents to one end.
- Product images in a hero bleed off an edge (show 60-90% of the window, fill at least two thirds of the width) rather than floating whole on a plate with even margins. Crop filler, never words or actions.
- Bento grids: one 2-span hero tile, a third to a half of tiles show real UI, neighbours differ in kind, no string repeats. Stats 48-72px with number and unit on one line.
- Secondary objects (a receipt, a phone, a detail card) straddle the main surface's edge and cover chrome, not content.
- Decide density by usage: tools used all day are dense and keyboard-first; occasional tasks are obvious and minimal.

### Patterns that work

- **Sequence as hero.** When the subject is a journey (a trip, a pipeline, onboarding, a project plan), make the sequence itself the hero: place or phase names large on a drawn line, the user's next step highlighted, a live countdown or "next up" line.
- **Chapters.** A sticky left column holds the chapter's big title and its context; the right column holds its content. Works for legs of a trip, phases of a plan, sections of a report.
- **People are faces.** Lead with large portrait tiles (4:5) for the people who matter, compact rows for the long tail. With no photo, show initials large in the section colour, not a grey silhouette.
- **Brief layout.** Portrait + name in display type; an at-a-glance strip of three facts (when, why it matters, what we want); the actions or talking points; one contrasting panel for the recommendation; evidence and sources in a quiet band at the end.

## Typography

- One family, or two that clearly contrast (for example a condensed display face used large as the visual, with a sturdy grotesk for body). Avoid reflex defaults; check the fonts are actually loaded.
- Self-host woff2, `font-display: swap`, preload only the display face. Use metric-matched fallbacks to avoid layout shift.
- A real scale (for example 14/16/20/28/44/88) and one weight step for hierarchy. No more than three sizes in one view of product UI.
- Body 16px or more, line-height 1.4-1.6, 45-75 characters per line; serif body gets slightly more leading. UI strings 40-60 characters.
- Tighten tracking slightly on very large display type.
- `text-wrap: balance` on headings, `pretty` on short paragraphs.
- Tabular figures for columns, timers and counters that update; proportional figures for a single centred display number.
- Keep monospace for code, IDs and numbers that must line up; it is not a shortcut to looking technical.
- Never shrink text to fit more in. If it does not fit, cut content.
- Pick sample names and values whose length fits, so nothing truncates in the demo.

## Colour

- One neutral family with about five ink steps, one accent, and semantic status pairs (soft background + ink foreground for good, warn, bad, idle). Test each pair for contrast.
- The accent marks actions, the current location and brand moments only.
- Prefer near-black and off-white to pure #000 on #fff in UI; use true black or white only when intended.
- Dark mode is its own design: lighter surface steps instead of shadows, re-tuned accents and status pairs. Follow `prefers-color-scheme`.
- A filled button hovers one step darker or lighter within its own hue, never by swapping to grey.
- No translucent objects overlapping each other; icons are one opaque colour.
- Gradients and glows must serve the content. Subtle tinted depth is fine; gradient text, neon and coloured glows on cards are not.

## Surfaces, radius, shadow

- Radii are concentric: outer radius = inner radius + padding. Above about 24px of padding, choose each independently. Larger, more important surfaces can carry larger radii; pills for buttons are a choice, not a default.
- One shadow language with named levels (ground, card, raised, floating, modal). A layered ring works well for raised elements: a 1px ring at about 6% opacity, a short lift at 6%, an ambient blur at 4%; deepen slightly on hover. Dark mode: a single white ring at 8-13%.
- Dividers, table lines and input outlines stay as borders. Drop any border that the fill already provides.
- Images get a 1px inset outline at 10% (black in light, white in dark) so they do not bleed into the page.
- Keep status dots, badges and focus rings within the element's edge, with breathing room from their neighbours.

## Icons and imagery

- One icon set, one stroke weight, one size (14-16px in UI); use filled variants at 14px and below.
- Real companies get their real logo. Never draw a letter in a square as a stand-in.
- Imagery is a real photo, real product content, a clearly labelled draft, or a briefed placeholder. Never CSS or SVG scenes pretending to be photographs.

## Motion

Two registers:

| Register | Duration | Easing |
|---|---|---|
| UI feedback (hover, press, toggles) | 120-200ms | ease-out, e.g. `cubic-bezier(.2,0,0,1)` |
| Surfaces (menus, sheets, panels) | 200-300ms | ease-out; exits about half as long, ease-in |
| Full-screen or celebratory moments | 400-500ms | a gentle spring or `cubic-bezier(.2,.75,.2,1)` |
| Storytelling and demo animation | 0.5-1s per beat | travel `cubic-bezier(.45,0,.1,1)`, arrival `cubic-bezier(.25,.8,.25,1)` |

- Use transitions for interactive state (they can be interrupted and retarget from the current value); keyframes only for one-shot sequences.
- Stagger in reading order: 40-80ms between siblings, 80-120ms between tiles or sections, about 80ms per word for a split headline; total entrance under about 1s. Exits reverse and finish within 0.5s.
- Nothing pops in or vanishes: keep elements mounted through their exit; neighbours make room smoothly.
- Change only what changed: roll the changed digit, morph the same button's label, rotate the plus into a cross. Cross-fade (with a little scale and blur) only between unrelated icons.
- Solid objects arrive by moving, opaque; they do not ghost in. Landings decelerate without bounce. Overshoot only on small expressive icons.
- Elements already at rest do not animate on first load, except one meaningful entrance.
- The first frame and the reduced-motion frame must tell the whole story on their own.
- Loops (for demos): 10-16s, 3-5 beats, hold the payoff about 2s, reset out of sight or with a visible cause.

## Performance

- Animate transform, opacity, filter and clip-path only. List properties explicitly; never `transition: all`.
- Core Web Vitals targets: LCP under 2.5s, INP under 200ms, CLS under 0.1. Check current thresholds at web.dev before quoting them.
- Set dimensions on images and media, reserve space for injected content, preload the hero image and critical font, defer non-critical JS.
- `will-change` only on elements that are actually animating and only for compositor properties.
- CSS first. Reach for an animation library only for timelines, scroll-linked effects, runtime control or computed values; use whatever the project already has.
- Pause off-screen animations and clean them up on unmount.
