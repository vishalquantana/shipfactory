# QA: look, critique, fix

## The loop

1. Render the real thing. Screenshot at the shipped width and at 390px; light and dark separately if both ship. For motion, step through frames (40-80ms) at presses, morphs and transitions. For decks and PDFs, render every page to an image.
2. Review it as a picky senior designer would. Imagine them circling one stray detail on a screenshot and asking why it is there; whatever you imagine them circling is your next fix.
3. Fix in this order: framing and crop, then hierarchy and density, then states and copy, then motion, then polish. Biggest visible fault first. Re-shoot after each round.
4. For an independent read, give a reviewer only the screenshots and this checklist, not your reasoning.
5. Report honestly: what you checked, what you fixed, and what you did not check (devices, widths, states, screen readers). An automated pass is not approval.

## Critique format

For each finding: the problem, the principle it breaks, the effect on the user, the fix. Rate severity P0 (blocks use) to P3 (polish), and effort S/M/L. Open with what already works; merge repeated defects into one system-level fix.

## Checklist

1. The purpose and the primary action are obvious within three seconds. One focal point; the squint test passes.
2. Hierarchy survives grayscale. Nothing relies on colour alone.
3. Spacing, radius, shadow, type size and colour all come from one scale or token set.
4. Related items sit closer than unrelated ones. Edges align exactly. No repeated identical-card template.
5. Type is deliberate: at most three sizes per view of product UI, body 16px+, comfortable line length, fonts actually loaded.
6. Every interactive element has hover, visible focus, active and disabled states.
7. Loading, empty, error and success states exist; long text truncates gracefully.
8. Every action gets feedback within about a second; destructive actions have undo or a specific confirmation.
9. Contrast: text 4.5:1, large text and UI parts 3:1, in both themes.
10. Keyboard pass: logical order, no traps, focus never hidden, dialogs trap and return focus, Esc closes.
11. Semantic HTML, labelled inputs, alt text, `lang`, skip link, live regions for status.
12. Targets at least 24px, 44px for touch and primary actions.
13. Holds at 320px wide and 200% zoom without horizontal scrolling.
14. Copy: verb + object buttons, consistent terms, useful errors, no filler or placeholder names.
15. Motion has a cause, uses transform and opacity, mostly 300ms or less, respects reduced motion.
16. Charts: right form, finding as the title, zero-based bars, one y-axis, consistent scales, validated palette, legend plus direct labels.
17. No tells from `tells.md`.
18. Client-facing: no internal notes, team names, caveats or wrong branding anywhere in the rendered artefact.
19. Private material: password-gated and noindexed; no personal contact details published.
20. Rendered and inspected: no overlap, clipping, overflow or collisions.
