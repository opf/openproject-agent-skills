# Baseline scenario (RED)

The reference workflow for this skill is authoring a new reusable Primer-based
ViewComponent in the OpenProject repo: a generic primitive (`.rb` + `.html.erb`
+ Lookbook preview + docs + component spec) with a header (title, count badge,
primary action, overflow menu), repeated caller-supplied rows, an empty state,
and a visual scheme/variant option.

This file captures how a competent agent fails that workflow when the skill is
**not** loaded. The scenario is deliberately hard: the agents were told they
**could read the codebase** (`app/components/**`, `lookbook/**`, `spec/**`) and
match conventions — so these are not "didn't know the patterns" failures, they
are "read the patterns and still diverged" failures. That is what justifies a
skill over a link to the reference files.

## Setup

- Loaded skills: none specific to Primer/ViewComponents.
- Repo: an `openproject` checkout with `BorderBoxListComponent`,
  `WidgetBoxComponent`, the Primer gem, `CLAUDE.local.md`, and the shared spec
  examples all present and readable.
- Task: two near-identical component briefs (`SummaryPanelComponent`,
  `SectionCardComponent`) run as independent agents, to surface variance.

## Failure modes observed

Two independent agents (A = SummaryPanel, B = SectionCard), both with full
codebase access:

### F1 — Enum representation diverges

The codebase shows two idioms: a stored validated symbol (Primer upstream) and
the `ActiveSupport::StringInquirer` wrap (the accepted house idiom, introduced
in `BorderBoxListComponent`). Agent B used `StringInquirer`; Agent A stored a
plain symbol, justified by happenstance ("only two values, checked in one or two
spots") rather than principle. Same input, opposite shape — and the two carry
different comparison semantics (`variant == :borderless` holds for the symbol
but not the `StringInquirer`, which compares as a `String`). The defect is the
**inconsistency**, not either idiom: the skill must prescribe one. It prescribes
the house `StringInquirer` idiom and documents the `String`-vs-`Symbol` gotcha.

### F2 — Two spellings for one attribute

Agent B exposed the overflow menu's accessible label **two ways** at once —
`button_aria_label:` *and* `button_arguments: { aria: { label: } }` — a
deprecated-on-arrival API with no single obvious call. (This is also the latent
shape of the repo's own `HasMenu`, so "match conventions" actively pulls toward
it.)

### F3 — Boolean named for mechanism, not effect

Agent A shipped `interactive:` whose sole job is to add `aria-live="polite"`.
Opaque at the call site; a reader cannot tell what `interactive: true` does.

### F4 — `render?` dead code

Agent A wrote `render? = rows.any? || empty_state? || true` — the trailing
`|| true` makes the guard always true, defeating its purpose.

### F5 — Specs hand-rolled, shared examples ignored

Both agents wrote bespoke assertions and used **none** of
`spec/support/shared/components/` (`rendering Box`, `rendering an empty Border
Box List`, `a reorderable Border Box List`), duplicating coverage and risking
tautological tests.

## Not exercised greenfield (still encoded)

The heading-accessibility footgun (interactive content inside a heading slot)
and the migration-preservation losses (drag-and-drop attrs, empty-collection
guards, locale keys) did not fire because the task was greenfield authoring, not
migration. They are high-severity during migration, so the skill encodes them as
checklist items in `migrating-consumers.md` regardless.
