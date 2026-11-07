# Decks and briefs

Practices from building client decks and private meeting briefs from structured data.

## Decks

- One idea per slide, readable from across a room. If a slide needs a paragraph, it is two slides or a handout.
- A reliable spine for a tailored deck: cover (their name, their problem in one line) → where we can help (three specific ways) → how we start (a small, concrete first step) → evidence (real screens, real results) → proposed next step.
- Brand by audience. A white-label or partner deck carries only the partner's name and logo, including inside screenshots; swap out any image that shows your own name.
- Evidence beats adjectives: a real product screen with three short points beats a page of claims.
- Generate decks from data, then render every page to an image and look at all of them. Generators produce overflow, orphaned words and duplicated text that only show up when rendered.
- Check the cover line for leftover data artefacts: brackets, "(since 2026)", "unknown", duplicated organisation names.

## Keep internal thinking out of client material

Research produces two kinds of text: facts the client may see, and our reasoning about the client. Keep them apart in the data model, not just at render time.

- Fields like "why this slide fits", "rapport hook", "our ask", "avoid", "confidence" and "sources" are internal by definition. Never render them onto a client slide.
- Filter generated lines that mention your own team, "unverified", "confirm", "ask X for context" and similar.
- After building, search the artefact itself (the HTML, the text inside the PPTX/PDF) for those words. Fix the generator, not the output.
- Do not rely on "the client will not read the small print". The subtitle and the footer get read.

## Private briefs (people, accounts, deals)

- Lead with the person: a large portrait, the name in display type, their role, and a one-line summary of who they are.
- An at-a-glance strip of three facts: when we meet, why they matter or why we were introduced, what we want from the meeting.
- "Open with": five specific questions or openers in your own voice, ordered as you would use them.
- "Personal ground": public, respectful rapport notes (alma mater, career path, causes, recent talks or writing) and common ground. Nothing intrusive: no health details, minors, home addresses or private family matters.
- A short "steer clear of" line for sensitivities (politics, past controversies, competitors).
- The recommendation in one contrasting panel: headline, three ways to help, the first step.
- Background, organisation snapshot, recent news, sources and a confidence note in a quiet band at the end. Mark anything unverified as unverified.
- Show what you will present: the actual tailored deck as a filmstrip with download links, not generic library slides.
- When identity is uncertain (common names, no public footprint), say so plainly and do not invent a photo or a biography. A wrong photo is worse than initials.

## Publishing private material

- Password-gate it, and serve it with `X-Robots-Tag: noindex, nofollow`, a `robots.txt` that disallows everything, and a robots meta tag on every page.
- Keep phone numbers, personal emails and home addresses in a separate private file that is never published.
- Prefer a password-only form with a long-lived HttpOnly cookie over browser basic-auth dialogs for non-technical users.
