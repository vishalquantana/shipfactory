# Shipshape

One design skill for AI agents that build things people look at: web pages, product UI, dashboards, charts, slide decks and briefs.

It combines the best ideas from a dozen well-known design skills (credited in `references/credits.md`) with practices from Quantana's own products, rewritten as one coherent set of rules:

- a short process: brief, read the existing system, plan tokens, diverge, check for defaults, build, screenshot, critique
- 22 core rules covering hierarchy, type, colour, surfaces, states, motion, writing, accessibility and data
- how much boldness each surface gets (marketing vs product UI vs dashboards vs decks)
- client-safe output rules so internal notes never reach a client
- a catalogue of generated-design tells to avoid
- a 20-point QA checklist built around looking at the rendered result

## Install

Claude Code (personal):

```bash
mkdir -p ~/.claude/skills && cp -r skills/shipshape ~/.claude/skills/
```

Any agent that supports the skills CLI:

```bash
npx skills add vishalquantana/shipfactory --skill shipshape
```

Then ask your agent to build or review a page, deck or dashboard; it loads the skill when the task matches. You can also invoke it directly with `/shipshape`.

## Files

| File | What it covers |
|---|---|
| `SKILL.md` | The process, core rules, surface scoping, client-safe rules, the ban list |
| `references/craft.md` | Layout, type, colour, surfaces, icons, motion and performance numbers |
| `references/ux.md` | Framing, states and feedback, writing, WCAG 2.2 AA, design systems, ethics |
| `references/dataviz.md` | Chart choice, honesty, colour roles and validation, labels, interaction |
| `references/decks-and-briefs.md` | Tailored decks, keeping internal notes out, private briefs |
| `references/tells.md` | The generated-design tells, grouped |
| `references/qa.md` | The look-critique-fix loop, critique format, checklist |
| `references/credits.md` | Sources, licenses and install commands for the originals |

MIT licensed.
