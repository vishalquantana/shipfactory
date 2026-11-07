# Credits

Shipshape is original writing. It distils ideas, not text, from the skills below and adds practices from Quantana's own products. Nothing here is copied verbatim. For the full depth of any area, install the original.

| Skill | Author / source | License | What we learned from it |
|---|---|---|---|
| frontend-design | Anthropic, `claude-plugins-official` | Apache-2.0 | Plan tokens before code, the default check, boldness in one place, the catalogue of generated-design tells, writing as design |
| make-interfaces-feel-better | Unattributed in our copy; appears to build on Jakub Krehel's essay on interface details | Not stated | Concentric radii, press scale, layered ring shadows, image outlines, never `transition: all` |
| stageshow | Benji Taylor, github.com/benjitaylor/skills | MIT | Framing and cropping, spacing numbers, motion registers and easing, word budgets, the render-and-critique loop |
| web-perf | Cloudflare skills | Not stated | Core Web Vitals targets, layout-shift prevention, loading priorities |
| gsap-core, gsap-performance | GreenSock, github.com/greensock/gsap-skills | MIT (stated in skill metadata) | Compositor-only animation, cleanup, reduced motion, when a library is worth it |
| design-an-interface | Matt Pocock (marked deprecated upstream) | Not stated | Diverge into three or more concepts, then converge |
| design-critique, design-elevation, interaction-design, ux-writing, accessibility-audit, design-systems, ux-strategy | Carlos Cuellar, design-skills plugin | MIT | Critique method, states and timing, UX writing patterns, WCAG 2.2 checks, token architecture, outcome framing |
| design, design-review | Dimitrios Arapis, design-superpowers plugin | MIT | Composing from existing systems, review checklist, AI-tell detection |
| canvas-design | Anthropic example skills | Apache-2.0 | Reduce before adding; look at the rendered output |
| dataviz | Bundled with Claude Code | Not stated | Form before colour, palette roles, numeric palette validation, chart honesty |

## Install the originals

```bash
# stageshow (Benji Taylor)
npx skills add benjitaylor/skills --skill stageshow

# GSAP skills (GreenSock)
npx skills add greensock/gsap-skills

# frontend-design (Anthropic) - in Claude Code
/plugin install frontend-design@claude-plugins-official
```

Other plugins above are available from their authors' marketplaces.

## Quantana's own contributions

The surface-scoping table, "the accent is a promise", honest progress, celebrate effort, the moments-and-emotions journey table, benchmarking against consumer apps, the sequence-as-hero, chapter and people-as-faces patterns, the brief layout, and all of the client-safe output rules come from Quantana product and sales work in 2026.
