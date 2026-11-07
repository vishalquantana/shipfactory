# UX: states, writing and accessibility

## Frame the work

- Name the outcome the screen serves as a user behaviour you could measure, not a feature.
- Map the user's day as moments: a table of moment, job to be done, emotional state, what the product does today, what it should feel like. Design for the emotional state as well as the task.
- Benchmark against the best consumer products in the user's world (delivery trackers, payment success screens, reward unlocks), not against other internal tools. Score honestly out of 10 and say what would move it.
- Compare the first prototype with what shipped. Personality often gets lost in the build (a greeting, a comparison with the user's own average, a celebration). Restore what was intended.
- Prefer familiar patterns to novel ones; work from the user action to the component, not the reverse.
- Limit each view to about three primary decisions; past about seven options, group or filter.
- Never surprise people mid-task: no permission prompts, alerts or loud motion while they are doing the real-world job. Ask and prime up front.

## States and feedback

- Every interactive element: rest, hover, keyboard focus, pressed, selected, disabled, loading, empty, error and success.
- Respond within 100ms; show progress at 1-10s; move longer work to the background and notify when done. Never leave more than a second of silence.
- Skeletons match the final layout. On refetch, keep the old content dimmed instead of flashing a spinner.
- Undo for reversible actions. Confirm only irreversible ones and name the consequence and count.
- Prevent errors with constrained inputs, sensible defaults and validation on blur.
- Avoid modes; if one is unavoidable, make it unmistakable.
- Every gesture or drag has a visible alternative. Do not override platform gestures.
- Celebrate effort as well as outcome (progress, streaks, a hard task handled). People who are struggling are the ones most likely to give up.
- Design the end of each flow: a clear success state and the next step.

## Writing

- Buttons: verb + object, sentence case, 2-4 words. The same action keeps the same name (Publish, then "Published").
- Errors: what happened, why, how to fix it. No codes on their own, no blame, no dead ends. Inline errors name the field and the rule.
- Empty states: why it is empty plus the action that fills it. Success: brief past tense.
- One term per concept across the product; specific labels ("Invoices", not "Resources").
- Plain language: key information first, active voice, "you", sentences of about 8-14 words for public audiences.
- Match tone to stakes: no jokes in errors, security or payments.
- Unambiguous dates ("5 Aug 2026"). Sentence case by default.
- No filler or slogan words, no placeholder names, no invented numbers unless clearly marked as a mock.
- Link text makes sense out of context.

## Accessibility: WCAG 2.2 AA floor

| Area | Requirement |
|---|---|
| Structure | Semantic HTML first: buttons, links with href, headings in order, landmarks, lists, tables, labelled inputs. ARIA only when nothing native fits. |
| Contrast | Text 4.5:1, large text 3:1 (1.4.3). UI parts, focus rings and chart marks 3:1 (1.4.11). Placeholders too. |
| Colour | Never the only signal (1.4.1). Hierarchy survives grayscale. |
| Keyboard | Everything operable (2.1.1), no traps (2.1.2), logical order (2.4.3), visible focus (2.4.7), focus not hidden under sticky UI (2.4.11). |
| Dialogs | Trap focus, close on Esc, return focus to the trigger. Disclosures expose `aria-expanded`; icon buttons have an accessible name. |
| Targets | At least 24px (2.5.8). Use 44px for touch-first and primary actions as best practice. Drag has an alternative (2.5.7). |
| Reflow | Works at 200% zoom (1.4.4), at 320px wide (1.4.10) and with user spacing overrides (1.4.12). |
| Forms | Visible labels, never placeholder-as-label, required marked without colour alone, `autocomplete` for personal data, no re-asking for data already given (3.3.7), no puzzle-only login (3.3.8). |
| Announcements | Status changes via `aria-live` or `role=alert` (4.1.3). Page `lang` set (3.1.1). Skip link (2.4.1). Help in a consistent place (3.2.6). |
| Media | Meaningful alt text; decorative images `alt=""` (1.1.1). Nothing flashes more than three times a second. |
| Motion | Honour `prefers-reduced-motion`: remove movement and parallax, keep fades and the final state. |

## Design systems

- Components reference semantic tokens (global, then semantic, then component), never raw hex or px. Themes swap token values only.
- Name tokens and components by purpose ("action-primary", "Card"), not appearance.
- A component spec covers anatomy, at most about five variants, all states, keyboard and ARIA behaviour, content limits and responsive rules.
- A missing component is a gap to log, not a one-off override. Promote a pattern once three or more places need it.

## Ethics

No dark patterns: no confirm-shaming, false urgency, hidden costs or pre-ticked consent. Ask who is excluded or harmed by the design.
