---
name: writing-primer-viewcomponents
description: Use when authoring, reviewing, or migrating a reusable OpenProject ViewComponent built on GitHub Primer (Primer::Beta::*, Primer::Alpha::*, or OpenProject Primer variants) — designing a slotted primitive's slot/enum/system-argument API, moving bespoke markup onto a shared component such as BorderBoxListComponent, or writing its Lookbook preview, docs, or component spec. Applies in app/components/** and modules/*/app/components/** of the openproject repo.
---

# Writing OpenProject Primer ViewComponents

## Overview

OpenProject builds reusable UI as ViewComponents that wrap GitHub Primer
(`Primer::Beta::*`, `Primer::Alpha::*`, and the OpenProject Primer variants).
This skill is the team's house style for authoring those components, migrating
bespoke markup onto shared ones, and speccing them.

**Core principle: match the conventions that already exist; do not invent new
ones where one exists.** The named gold standard is
`modules/grids/app/components/grids/widget_box_component.rb`. The most-copied
list primitive is `app/components/open_project/common/border_box_list_component.rb`
(+ its subdirectory) — strong overall, but it carries a few legacy smells called
out below, so read it as *both* example and anti-example.

Prefer small, well-tested slotted primitives + a Lookbook preview + a component
spec over large multi-file PRs. Scoping consumer rewrites out of a primitive PR
is fine.

### "Match conventions" means the recipes below — not the legacy smells

`BorderBoxListComponent` is the most-copied primitive, so its mistakes spread.
**Matching house style means following the recipes in this skill, not
replicating these specific smells** even though they ship in that component
today. Do **not** carry any of these into a new component:

- a menu trigger with **two** label spellings — `button_aria_label:` *and*
  `button_arguments: { aria: { label: } }` (expose only `button_arguments:`)
- an `interactive:` boolean for an aria-live effect (name it for the effect)
- a domain model baked into a generic primitive (`with_work_package_item`)

"But the existing component does it" is not a reason to repeat it. For these three
points, `BorderBoxListComponent` is the anti-example. (Its
`StringInquirer`-wrapped enums are **not** on this list — that is an accepted
house idiom; see the enum recipe below.)

## When to Use

- Authoring a new reusable Primer-based ViewComponent (a primitive)
- Designing or reviewing a component's slot / enum param / system-argument API
- Migrating hand-written ERB markup onto a shared component (e.g. a list box →
  `BorderBoxListComponent`)
- Writing the Lookbook preview, docs page, or RSpec component spec for one

Not for: one-off page-specific partials, non-Primer markup, or domain logic.

## Prerequisites (repo-aware)

This skill assumes you are working inside an `openproject` checkout. Read these
convention sources before writing — quote/link them, do not restate:

- `CLAUDE.local.md` sections **"## Primer ViewComponents"**, **"### List vs table
  components"**, **"## Spec Style"**, **"### Component specs"**, **"## Commit"**
- `app/AGENTS.md` (a.k.a. `app/CLAUDE.md`): UI strings via translation keys
  (never hard-coded), YARD docs, Lookbook previews, `erb_lint` before commit
- Gold standard: `modules/grids/app/components/grids/widget_box_component.rb`
- Reference family: `app/components/open_project/common/border_box_list_component.rb`
  and `app/components/open_project/common/border_box_list_component/`
- Docs/preview/spec exemplars: `lookbook/docs/components/border-box-list.md.erb`,
  `lookbook/previews/open_project/common/border_box_list_component_preview.rb`,
  `spec/support/shared/components/`, `modules/meeting/spec/components/meetings/table_component_spec.rb`

For upstream Primer conventions, read the installed gem — find it with
`bundle show openproject-primer_view_components` (do not hard-code a version
path), then read `app/components/primer/beta/border_box.rb`,
`app/components/primer/beta/counter.rb`, `app/lib/primer/attributes_helper.rb`.

## The component skeleton

Subclass `ApplicationComponent`. Include `OpPrimer::ComponentHelpers` and
`Primer::FetchOrFallbackHelper`. Every new file needs `# frozen_string_literal: true`
and the GPL copyright header (copy from any sibling). Document with YARD.

```ruby
class FooComponent < ApplicationComponent
  include OpPrimer::ComponentHelpers
  include Primer::FetchOrFallbackHelper

  def initialize(title:, scheme: SCHEME_DEFAULT, **system_arguments)
    super()
    @title = title
    @scheme = fetch_or_fallback(SCHEME_OPTIONS, scheme, SCHEME_DEFAULT)
    @system_arguments = system_arguments
    @system_arguments[:id] ||= self.class.generate_id
    @system_arguments[:classes] = class_names(@system_arguments[:classes], "op-foo")
  end
end
```

## API design recipes (the heart)

Get these shapes right; the rest is mechanical. Full worked example + YARD,
`merge_aria` semantics, and keyword-arg ordering live in
[authoring-primitives.md](authoring-primitives.md).

**Enum param.** Define `FOO_DEFAULT` and a frozen `FOO_OPTIONS` (default first);
validate with `fetch_or_fallback(FOO_OPTIONS, value, FOO_DEFAULT)`. The house
idiom (from `BorderBoxListComponent`) wraps the validated value in
`ActiveSupport::StringInquirer` so you branch with predicate methods:

```ruby
@scheme = ActiveSupport::StringInquirer.new(fetch_or_fallback(SCHEME_OPTIONS, scheme, SCHEME_DEFAULT).to_s)
```

Then `@scheme.condensed?`, and drive CSS via a
`class_names(..., "op-foo_x" => @scheme.x?)` hash. **One gotcha:** a
`StringInquirer` is a `String`, so a public `attr_reader :scheme` compares as
`scheme == "condensed"` / `scheme.condensed?`, **not** `scheme == :condensed`.
If a component instead needs to expose the value for external *symbol*
comparison, store a plain symbol there and branch with `==`/`FOO_MAPPINGS`.
Either way, pick one shape per component and keep it consistent.

**Slots.** `renders_one`/`renders_many`, lambda forwards `**system_arguments` to
the wrapped Primer component. Polymorphic rows use
`renders_many :items, types: { item:, work_package_item: }` (auto-generates
`with_item` / `with_work_package_item`). Keep names coherent (`with_action_button`
/ `with_action_icon_button`). **One concept → one name:** don't store a `title:`
kwarg as `@title_text` while a `with_title` slot and a `title_content` reconciler
fight over the same word.

**System arguments.** Accept `**system_arguments`, forward verbatim to the
wrapped Primer component, let Primer own its defaults. Only augment `:classes`
(via `class_names`, caller classes preserved), `:id`, and your own knobs. Don't
re-document a forwarded Primer arg as if you own it.

**`before_render` vs `initialize`.** Pure derivation → `initialize`. Anything
that depends on captured slot content → `before_render`, which must call
`content` first to force slot capture, then post-process (auto-fill an empty
state, resolve a count from `items.size`, wire `aria-labelledby`). Validate
required content (e.g. "title kwarg OR `with_title` slot") in `before_render`,
**not** `initialize` — slots aren't known yet at init.

**`render?` ordering.** ViewComponent runs `before_render` **before** `render?`,
so a default slot populated in `before_render` is visible to `render?`. Gate
`render?` on real presence — `header? || items.any? || empty_state? || footer?` —
never `... || true` (that is dead code that defeats the guard).

**One obvious way.** No two spellings for one attribute in a new API (ship
`button_arguments: { aria: { label: } }` **or** `button_aria_label:`, not both).
**Name booleans for their effect:** `announce_changes:`, not `interactive:`,
when the only job is `aria-live="polite"`.

## Accessibility: headings hold phrasing content only (high severity)

A title rendered via `Primer::Beta::Truncate(tag: :h4)` emits `<h4>…</h4>`.
Headings allow **phrasing** content only. A single `<a>`/`<span>` is fine; a
`flex_layout`/`<div>`, a `<button>`, a counter+button cluster, a menu, or a
dialog inside the heading is **invalid HTML** and collapses the whole header into
one heading for screen readers.

Put title text/link in the title slot; route everything else through sibling
slots (`with_action_button`, `with_action_icon_button`, `with_menu`,
`with_description`). Never fold rich/interactive header content into `with_title`.
(`CLAUDE.local.md` states this rule — cite it.) This bites hardest during
migration — see [migrating-consumers.md](migrating-consumers.md).

## Migrating bespoke markup onto a shared component

List-vs-table choice and the verbatim-preservation checklist (drag-and-drop
attrs, turbo frames, locale keys, accessible names, empty-collection guards) are
in [migrating-consumers.md](migrating-consumers.md). Read it before moving any
existing markup onto `BorderBoxListComponent` or similar.

## Specs

Every touched/added component gets a spec. Reuse the highest-level shared example
that fits; assert structure + semantics, not bare presence; keep it
non-tautological. Patterns, matcher vocabulary, and gotchas are in
[component-specs.md](component-specs.md).

## Anti-patterns (and why)

| Anti-pattern | Why it's wrong | Instead |
|---|---|---|
| Two spellings for one attribute (e.g. copying `HasMenu`'s `button_aria_label:` *and* `button_arguments:`) | Deprecated-on-arrival; ambiguous API; "it's the existing convention" is not a defense — it's the named anti-example | One obvious way (`button_arguments: { aria: { label: } }`) |
| `interactive:` / `mode:` boolean | Opaque at call site | Name for effect: `announce_changes:` |
| `render?` ending in `\|\| true` | Dead code; the guard never fires | Gate on real slot/content presence |
| Validating slots in `initialize` | Slots aren't captured yet → false negatives | Validate in `before_render` after `content` |
| Interactive content inside `with_title` | Invalid HTML; collapses heading for AT | Title text only; actions via sibling slots |
| Hand-maintained `delegate` allowlist on a Primer wrapper | Silently drops the rest of the wrapped API; rots on upgrade | Delegate broadly, or document a deliberately narrow facade |
| Domain model baked into a generic primitive | Layering inversion (generic list knowing `WorkPackage`) | Generic `with_item`; the domain consumer composes the domain row |
| Documenting a forwarded Primer arg as owned | Drifts from reality (see the `padding:` row in the BorderBoxList doc) | Document only knobs your component owns |
| Hand-rolled spec assertions | Tautological; misses shared coverage | Reuse the highest-level shared example |

## Checklist

Create a todo per item.

- [ ] Read the convention sources above; confirm no existing primitive already
      fits (list-shaped → `BorderBoxListComponent`; columns → a table)
- [ ] `frozen_string_literal`, GPL header, `ApplicationComponent`,
      `OpPrimer::ComponentHelpers`, `Primer::FetchOrFallbackHelper`
- [ ] Enum params: `FOO_DEFAULT` + frozen `FOO_OPTIONS`, `fetch_or_fallback`,
      one consistent stored shape (house `StringInquirer` idiom, or a plain
      symbol where external `== :sym` comparison is needed)
- [ ] Slots forward `**system_arguments`; one concept → one name; coherent
      polymorphic `types:`
- [ ] Slot-dependent work (validation, counts, empty state, `aria-labelledby`)
      in `before_render` after `content`; `render?` gates on real presence
- [ ] One spelling per attribute; booleans named for effect
- [ ] No interactive/flow content inside any heading slot
- [ ] UI strings via translation keys; YARD with `<%= one_of(...) %>` and
      `<%= link_to_system_arguments_docs %>`; `@!parse` slot signatures include
      `&block`
- [ ] Lookbook preview + docs page consistent with each other and the real API
- [ ] Component spec reuses shared examples; structural + accessible assertions;
      non-tautological
- [ ] `erb_lint` clean before commit
