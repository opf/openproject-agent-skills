# Migrating bespoke markup onto a shared component

When you move hand-written ERB onto a shared primitive (usually
`OpenProject::Common::BorderBoxListComponent`), the risk is silent loss — of
behavior, accessibility, or test coverage. Decide the target shape first, then
preserve every load-bearing detail verbatim.

## List vs table (decide first)

From `CLAUDE.local.md` "### List vs table components":

- **List-shaped** `BorderBox` — a header plus repeated **single-content** rows →
  `OpenProject::Common::BorderBoxListComponent`. Map header → `with_header`,
  rows → `with_item` / `with_work_package_item`, empty → `with_empty_state`
  (a default auto-renders). **A name plus one control (a toggle, a drag handle)
  is still a list.**
- **Table-shaped** — rows that are **multiple aligned data columns** with
  column-label headers → a `BorderBoxTable` / `DataTable`, **never** the list
  component.

**Don't extend the shared header API to fit one consumer.** If a header needs
several menus plus a dialog, leave the shared component as-is or defer the
migration — don't bolt on slots for a single caller.

## The heading-accessibility footgun (high severity)

This is where the heading rule from `SKILL.md` actually bites. Bespoke list
headers often render an entire layout — a flex `<div>` with a counter, a
"mark all" `<a>` button, a menu — and the naive migration drops that whole blob
into `with_title`. The title slot renders inside a heading tag
(`Primer::Beta::Truncate(tag: :h4)` → `<h4>`), so the result is
flow/interactive content inside an `<h4>`: **invalid HTML**, and screen readers
collapse the entire cluster into one heading.

Correct decomposition:

- title text or a single link → the title slot
- counter / buttons / menus / dialogs → sibling slots (`with_action_button`,
  `with_action_icon_button`, `with_menu`, `with_description`)

If the shared component has no sibling slot for something in the old header,
that's a signal the migration isn't ready — not a reason to cram it into the
title.

## Preservation checklist — copy verbatim

Carry every one of these across unchanged, and grep to prove nothing dangles:

- [ ] **Drag-and-drop attrs:** `data-generic-drag-and-drop-target='container'`,
      `data-target-container-accessor=':scope > ul'`,
      `data-target-allowed-drag-type`, per-row `data-draggable-type` /
      `data-draggable-id`, `data-drop-url`
- [ ] **Turbo** frame / stream wrappers
- [ ] **Stimulus** targets and **filter** targets
- [ ] **Locale keys** (every UI string stays a translation key)
- [ ] **Accessible names** (button/link labels, `aria-label`s)
- [ ] **Test selectors** the existing specs rely on
- [ ] **Empty-collection guards.** This one is a trap: the shared list
      **auto-renders a default empty state**. An old `if collection.any?` guard
      that rendered *nothing* when empty will, after migration, silently start
      rendering a blank-slate box. Treat dropping any such guard as an
      intentional, called-out change.
- [ ] **Removed CSS classes/ids have zero remaining references** — grep
      `frontend/src`, `*.sass`/`*.scss`, `*.ts` before deleting them.

## Generic primitive, domain consumer

Don't bake a domain model into a generic primitive. A generic list that knows
`WorkPackage`, `current_user`, and `project` (as the `with_work_package_item`
path in `BorderBoxListComponent` does) is a layering inversion — useful to read,
but not a pattern to spread. A new generic primitive gets a generic `with_item`;
the domain consumer composes the domain row and passes it in.
