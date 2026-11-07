---
name: shipshape
description: One design skill for anything an agent builds that people will look at - web pages, product UI, dashboards, charts, slide decks, briefs and reports. Use it before writing UI code or generating a deck, when someone says a page "looks boring", "looks AI-generated", "feels off" or "make it look better", when reviewing a design, and before shipping anything client-facing. Covers the plan, the visual system (type, colour, space, surfaces, motion), interaction states, UX writing, accessibility (WCAG 2.2 AA), data visualisation, decks, and a screenshot-based QA loop.
license: MIT
metadata:
  version: 1.0.0
  author: Quantana (ShipFactory)
---

# Shipshape

Make what you build look intended: one clear idea per screen, a visual system you chose on purpose, every state designed, and nothing that gives away that a machine made it. Then prove it by looking at the rendered result, not the code.

## The one idea

**Decide what the viewer must get in three seconds, make that the only loud thing, and keep everything else quiet, consistent and correct.**

Most weak output fails in one of five ways:

- **Generic.** Choices that would appear for any brief: the same fonts, the same card grid, the same gradient. Nothing comes from the subject.
- **Flat.** Everything has the same weight. The eye has nowhere to land.
- **Busy.** Every row filled, every label shown, borders and badges everywhere.
- **Unfinished.** Missing states (loading, empty, error), voids, misaligned edges, clipped text.
- **Leaky.** Internal notes, placeholder names or research caveats end up in front of a client.

## Process (always)

1. **Brief in one breath.** Who is looking, what they are trying to do, what this screen's one job is, and the one thing they must take away. If the brief does not say, propose it and state your assumption.
2. **Find the existing system first.** Read the project's tokens, CSS variables, Tailwind config, components or DESIGN.md. Compose from them. Only invent values when none exist, and say they are provisional.
3. **Plan before code.** Write a compact token plan:
   - colour: 4-6 named hex values (one neutral family, about five ink steps, one accent, semantic status pairs)
   - type: 1-2 families and their roles, with a real scale
   - layout: an ASCII sketch of the hero and the main sections, with alignment
   - the focal element: the single thing that carries the boldness
4. **Diverge, then converge** (for anything new or important). Sketch three genuinely different directions under different constraints, compare them in two lines each, pick or combine.
5. **Default check.** For each axis of the plan ask: would a similar prompt produce exactly this? If yes, change it and note why. See `references/tells.md`.
6. **Build to the floor** (below), then **look**: screenshot at the real width and at 390px, in light and dark if both ship. Fix the biggest visible fault first, re-shoot, repeat. See `references/qa.md`.
7. **Content audit before anything client-facing.** Search the output for internal words and strip them. See "Client-safe output" below.

## Scope the boldness by surface

| Surface | Personality | Rules that dominate |
|---|---|---|
| Marketing page, hero, landing, poster, deck cover | A distinct point of view taken from the subject's own world | One memorable element, expressive display type, one orchestrated entrance |
| Product UI, app screens, tools | The product's own tokens, quiet and dense enough for the task | One hero per screen, all states, fast feedback, keyboard parity |
| Dashboards and charts | Recessive chrome, honest data | Form before colour, validated palettes, title states the finding |
| Decks and briefs | One idea per slide or block, readable from across a room | Client-safe content, audience branding, evidence over adjectives |

Details: `references/craft.md` (visual system and motion), `references/ux.md` (states, writing, accessibility), `references/dataviz.md`, `references/decks-and-briefs.md`.

## The core rules

Hierarchy and layout
1. **One hero per view.** One focal point and one primary action. Everything else is secondary, folded, or one tap away. Pass the squint test and the grayscale test.
2. **Boldness in one place.** One memorable element; everything around it disciplined. Before shipping, remove one accessory.
3. **Structure must mean something.** Borders, numbers, dividers and labels only where they encode information. Numbered markers only for real sequences.
4. **One spacing scale** (4/8 base). Gaps between groups are larger than gaps inside them. Edges align exactly or sit clearly apart. No voids, no rows spread to fill height.
5. **Compose, do not tile.** Vary section layouts down a page; important items get more space (a large portrait, a wide tile), the long tail gets compact rows. Identical cards in a grid read as a template.
6. **Every width is its own design.** Recompose for narrow screens; hold at 320px and 200% zoom.

Type and colour
7. **Choose type on purpose.** One family, or two that clearly contrast. Self-host, preload the display face. Hierarchy from a real scale plus one weight step. Body 16px+, 45-75 characters per line, sentence case.
8. **The accent is a promise.** Reserve the brand or accent colour for what you can do (primary action), where you are (active state) and brand moments. Information uses ink and semantic tones.
9. **Status colours come in pairs and never alone.** Each state gets a soft background plus a readable ink, always with a word or icon. Red and green mean only error and success.
10. **Contrast is not optional.** Text 4.5:1, large text and UI parts 3:1. "Quiet" never means "faint". Design light and dark separately and follow the system theme.

Surfaces and detail
11. **Nested radii are concentric** (outer = inner + padding); radius varies with hierarchy, not one value everywhere.
12. **One shadow language** with named elevation levels; a different shadow must mean a different height. Prefer a fill or surface step before a border; dark mode gets depth from lighter surfaces.
13. **One real icon set**, one stroke, one size. No emoji as icons, no hand-drawn glyphs, no invented logos. Real photos or real product content, never decorative filler.

Interaction and motion
14. **Design every state:** rest, hover, keyboard focus, pressed, selected, disabled, loading, empty, error and success. Hover and focus come as a pair.
15. **Feedback starts on input and is fast:** press scale about 0.97, hover and press 120-200ms ease-out, panels 200-300ms, exits shorter than entrances. Interactive transitions must be interruptible.
16. **Every motion has a cause.** Motion answers an action or shows what changed. At most one orchestrated entrance per page; never a fade-up on every section. Animate only transform and opacity, never `transition: all`. Respect `prefers-reduced-motion` by showing the finished state.
17. **Honest progress.** Show time, not invented percentages ("about 2 min"). Never fake speed, never go backwards; say what is happening and what the user can do meanwhile.
18. **Celebrate the peaks.** The moment a result lands, a milestone, the end of a task deserves a designed moment and a clear next step. Undo for reversible actions; a specific confirmation ("Delete 3 files?") only for irreversible ones. Replace native alerts with sheets, toasts or swipe-to-confirm.

Words
19. **Copy is design.** User vocabulary, active voice, verb + object buttons ("Download the deck", never "Submit" or "Click here"). An action keeps one name through the flow. Errors say what happened and how to fix it; empty states invite the next action.
20. **Say it once.** No filler, no buzzwords, no placeholder names, no fake-precise numbers.

Accessibility floor (WCAG 2.2 AA)
21. Semantic HTML first; real buttons and links; headings in order; labelled inputs; visible keyboard focus never hidden under sticky UI; targets at least 24px (44px for touch and primary actions); `lang` set; status changes announced.

Data
22. **Form before colour; honesty before beauty.** Pick the chart from the job, start bars at zero, never use two y-axes, title the chart with its finding, assign colour by role and by entity (filters never repaint), validate the palette for colour blindness. See `references/dataviz.md`.

## Client-safe output

Anything a client, partner or the public will see must pass this before it ships:

- **No internal reasoning on the page or slide.** Strip research notes ("unverified", "rapport hook", "our ask", "why this slide fits"), team names, tool output, confidence caveats, and draft placeholders. Keep those on an internal page if you need them.
- **Audience branding.** White-label means no trace of your own name, including logos inside screenshots.
- **Grep the artefact itself** (HTML, the text inside a PPTX/PDF), not just your source data. Render every deck page to an image and look at each one.
- **Private material stays private.** Gate it behind a password, mark it noindex (header, robots.txt and meta tag), and keep phone numbers and home addresses out of published data.

## Never, unless the brief asks for it

The commonest tells of generated design, in one line each (full list in `references/tells.md`): purple-to-pink gradients, gradient text, glows and sparkle icons; cream background + serif + terracotta; near-black + one acid accent; the SaaS card kit (identical cards, one radius, one soft grey shadow); tracked ALL-CAPS eyebrows above every heading; "A · B · C" meta strings; an arrow on every button; one coloured word in the headline; 01/02/03 on things that are not steps; emoji bullets in a product UI; fade-up on every section; hover lift on every card.

If the brief explicitly asks for one of these looks, do what the brief says.

## Done means

You looked at the rendered result at desktop and mobile widths, in every theme shipped, ran the checklist in `references/qa.md`, fixed what spoiled the result, and listed honestly what you did not check.

## Credits

Shipshape is original writing that distils ideas from the skills listed in `references/credits.md`, plus practices from Quantana's own products. If you want the depth of any single area, install the original; the credits file has the commands.
