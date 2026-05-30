---
name: commit-message
description: >
  Writes intent-first commit messages — each body point names the problem this
  change solves, not what the diff shows. Use when generating any commit message,
  staging changes, or running `git commit`. Aggressively asks the user before
  writing when the why isn't visible in the diff; favors short sentences over
  paragraphs.
user-invocable: true
---

# Commit Message Style

## Audience

The reader is a developer hovering over a single line in inline git blame (Cursor / VS Code). They already see the code on screen — they know *what* it does. They want *why this line exists*, fast, before the hover dismisses. Treat every line of the commit body as something they read while their eye is on one source line; paragraph-length annotations cause mental fatigue and get skipped.

## Core Rule

Each point in the commit body answers "why was this needed?" — the problem it solves, the gap it fills, or the motivation behind it. Do not describe what was changed; the diff already shows that.

This style takes precedence over patterns observed in `git log`. Most of the project's commit history uses description-based messages. Follow the rules in this skill, not the historical pattern.

## What never goes in

The diff already shows these. Restating them costs reading time and adds zero intent:

- **Names visible in the diff** — file names, function names, identifier lists, enum members, locale codes. If the developer hovers and sees `header_capture_success`, they don't need the commit to name it back.
- **Counts of what changed** — "applied across all 10 locales", "updated 4 call sites", "renamed 3 files". The file list shows this.
- **Decompositions of a new name** — "(View + Purchase + Summary + Data)". The reader can parse the identifier.
- **Restating the new API surface** — naming the new helper signature or quoting a call expression that's right there in the diff.

## No implied premises

When a change "no longer needs X", "is now driven by Y", or means "no consumer change required" — the reader cannot verify the prior pain from your phrasing alone. Either:

- Name the prior failure concretely ("each new mode previously required editing the ternary in two screens AND adding a flat key in 10 locale files"), so the reader sees the gap that motivated the change, or
- Drop the point — vague positives ("cleaner", "easier to add new X", "now centralized") are filler that doesn't survive the litmus test.

If you find yourself writing a positive claim that depends on a prior-state premise the reader can't see, that's the signal to either back it up with specifics or to ask the user what motivated this.

## When implementation detail IS allowed

A brief description of *what* changed is appropriate in two cases:

1. **The code is genuinely complex or non-obvious from a first read** — e.g., a subtle state-machine transition, a non-local ordering invariant, a constraint imposed by a library's quirks. One short sentence orienting the reader before the why.
2. **A one-line description meaningfully helps the reader navigate the diff** — e.g., when the change is split across many files and a single shape sentence ("rotates the mutation key per re-focus to invalidate stale entries") helps the reader see the pattern.

Default is to omit. The exception is narrow; never use it to justify restating diff content.

## Format

```
type(scope): summary of intent (~76 chars, this is the IDE annotation)

1. Why this change was needed — the problem or gap
2. Why this change was needed — the problem or gap
...
```

- **Types**: feat, fix, chore, perf, refactor
- **Scopes**: repo-specific. Use the scope vocabulary documented in this repo's CLAUDE.md "Commit Convention" / "Commits" section. If none is documented, derive consistent scopes from recent `git log` rather than inventing new ones.
- First line: keep under ~76 characters — it appears as a single-line IDE annotation
- Body: numbered list, one point per distinct intent
- **Each point: 1-3 short sentences.** A point that became a paragraph is a smell — either you're padding (cut), or you're describing the diff (cut), or you're explaining a why that needs its own point (split).

## Process

When analyzing a diff to write the message:

1. Read the full diff, not just the file list
2. For each logical change, ask: "What problem did this solve for the user or developer?" Look at surrounding code context — comments, variable names, related changes in the diff, and nearby files that use the changed code.
3. For changes where the intent depends on context not visible in the diff, ask the user before writing that point. Two kinds of signal:
   - **Diff-content signals**: magic number or configuration changes, business logic where the "why" requires domain knowledge (device constraints, external specs, customer behavior), references to future work the diff doesn't address ("makes it easier to add X").
   - **Writing-process signals**: you find yourself about to write a vague positive ("cleaner", "easier", "now centralized"); you can't name the specific bug, feature ticket, design spec, or event that motivated this change; you're tempted to write "no longer needs X" but can't point to a prior failure where X bit someone. Each of these is the model telling you it doesn't know the why — ask, don't guess.

   Present what you observe changed and what makes the motivation ambiguous. If multiple changes need clarification, batch all questions together.
4. Group confirmed intents by theme, not by file or area changed
5. If a point only answers "what was changed," rewrite it to answer "why" — even for points informed by user answers

## Examples

Correct — leads with the problem, then briefly how it was addressed:

```
5. Pass passcode tokens on void/refund submissions. Before this, only
   route-based tokens were handled (via useProtectedQuery for GET
   requests), but void/refund are in-route capabilities, not navigable
   routes — so they had no token injection path for their POST mutations.
```

```
6. Resolve formatter conflict: ESLint and Prettier fought over trailing
   commas on save (e.g. secure/index.ts — one removed the comma, the
   other re-added it, causing visible flicker). Added eslint-config-prettier
   to disable ESLint's overlapping formatting rules, making Prettier the
   single source of truth.
```

Wrong — describes what was done, not why:

```
BAD:  "Set up Prettier with import organization"
BAD:  "Enforce passcode tokens on void/refund API calls via capability-based auth"
```

These describe the solution, not the problem. A future developer reading the annotation learns nothing about what motivated the change.

When intent depends on context outside the diff, ask before writing:

```
I see `staleTime` changed from `0` to `30_000` on the `useMerchantProfile`
query — this stops the profile refetching on every screen focus. What
motivated this specific value? (e.g., a flicker/perf issue, a rate limit,
a product decision that the data is fine to cache for ~30s?)
```

The user's answer ("the profile screen fired 3 requests per navigation and
tripped the rate limiter") produces a commit point no amount of diff-reading
could have.

Do NOT ask when the diff itself reveals the why:

```
Renaming `printEmvReceipt` → `printCardEmvReceipt` alongside a new
`printCardOnlineReceipt` — the rename clearly disambiguates card types.
No need to ask.
```

**Restating the diff — bad:**

```
1. Flat summary header keys (header_capture_success, header_preauth_success,
   header_success, header_void) required a new key and a new ternary arm for
   every mode added. With Increment joining the set, restructuring to a
   nested summary.header.{mode} object lets the consumer derive the key via
   t(`summary.header.${mode}`) — no consumer change needed when new modes
   are added. Applied across all 10 locales.
```

The key names, the new `t()` call, "Applied across all 10 locales" — all in the diff. "No consumer change needed when new modes are added" implies prior consumer changes were a pain but doesn't name the pain.

**Same intent — good:**

```
1. Each new mode used to need a hand-written flat key per locale plus
   a screen-side ternary arm. Restructured so the screen interpolates
   the key directly off the mode enum, matching the pattern already used
   for status labels in common.json. Adding a mode is now a locale-file
   entry, no screen edit.
```

States the concrete prior friction (hand-written keys + ternary arm per mode), the cross-codebase pattern this matches (status interpolation in `common.json`), and the future-work payoff. The reader doesn't need the keys re-listed.

**Implied-context — bad:**

```
4. ViewPurchaseSummaryData receives mode to support mode-specific field
   rendering on the summary card.
```

"Support mode-specific field rendering" is a vague positive that doesn't name what was wrong before. A reader hovering on the new prop sees `mode: ModeCollectionSummary` already; what they want to know is *why* pass it instead of deriving from `purchase`.

**Same intent — good:**

```
4. Differentiating mode-dependent UI from Purchase/EmvPurchase shape
   required mental gymnastics on status × auth_type × installment_plan
   combinations that drift as BE evolves. Threading mode from the parent
   (which knows it from the route) makes the screen's mode-branches
   greppable: Cmd+F "mode" shows every mode-dependent line and, by
   absence, every mode-independent one.
```

Names the concrete prior pain (BE shape drift, mental combinatorics) and the user-visible payoff (Cmd+F as a debugging affordance — something a developer maintaining the screen will actually do).

## Litmus Test

Before finalizing, re-read each numbered point and ask, in order:

1. Does this answer _why was this line changed_, or only _what changed_?
2. Is any part of it visible in the diff (identifier names, file counts, name decompositions)?
3. Does it imply a prior state the reader can't see ("no longer needs", "no X required", "now centralized")?
4. Is it 1-3 short sentences, or has it grown into a paragraph?

Rewrite — or cut, or ask the user — until every numbered point passes all four.
