# Loophole log

Refinements made to the skill after the GREEN scenario revealed a gap. Each
entry maps a discovered failure mode to the rule in `SKILL.md` that now carries
it.

## L1 — "Match conventions" was read as license to copy a legacy smell

**Discovered:** With the first cut of the skill loaded, a fresh agent building
`SectionCardComponent` exposed the overflow menu's accessible label two ways at
once (`button_aria_label:` *and* `button_arguments: { aria: { label: } }`),
re-creating the F2 baseline failure. Its stated reason: *"I matched the repo's
real Menu/HasMenu two-path label convention."*

The skill's core principle ("match the conventions that already exist; do not
invent new ones") and its "one obvious way" rule were in tension, and the agent
resolved it toward the existing — but legacy — `HasMenu` shape. The anti-pattern
table already listed "two spellings," but the agent did not classify the menu
label as an instance of it, because it perceived the dual path as an established
convention to honor rather than a smell to avoid.

**Fix:** Make the exclusion explicit. Added a dedicated block —
**"'Match conventions' means the recipes below — not the legacy smells"** —
naming the specific things in `BorderBoxListComponent` *not* to copy (the dual
menu-label spelling, the `interactive:` boolean, a domain model in a generic
primitive — but **not** its `StringInquirer` enums, which the maintainer
confirmed are an accepted house idiom), and stating outright that
"the existing component does it" is not a defense — for these points that
component is the anti-example. Sharpened the anti-pattern table row to name
`HasMenu` and `button_arguments:` directly.

**Skill section:** Overview → "'Match conventions' means the recipes below";
Anti-patterns table (two-spellings row).

**Re-test:** Two fresh agents, both pointed at `BorderBoxListComponent`/
`HasMenu`, then exposed exactly one menu-label spelling and explicitly cited the
anti-example when declining the second. Converged. See `with-skill.md` round 2.

## Notes on what was deliberately *not* turned into a loophole entry

- **Heading-a11y footgun** and **migration preservation** never failed in the
  greenfield GREEN runs (the task wasn't a migration). Rather than manufacture a
  scenario, they are encoded from the source handoff's real-PR evidence as
  checklist items in `migrating-consumers.md`. Flagged here so a future tester
  knows they are *unverified by pressure scenario* and should be exercised with
  an actual migration brief if stronger assurance is wanted.
- **Stale-info traps** surfaced by the reference-study pass (a forwarded
  `padding:` documented as an owned param; `with_description_content` vs
  `with_description`; include `AttributesHelper` only where you merge) were
  folded into the recipes/anti-patterns directly, so the skill does not teach the
  reference component's documentation bugs.
