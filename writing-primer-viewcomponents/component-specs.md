# Component specs

Every touched or added component gets a spec. Model on
`modules/meeting/spec/components/meetings/table_component_spec.rb`. Assert
**structure + semantics**, reuse the **highest-level shared example that fits**,
and keep every assertion **non-tautological**.

## Reuse shared examples — don't hand-roll

Shared examples live in `spec/support/shared/components/`. Use the most specific
one that applies; don't drop to a lower-level example or hand-write assertions a
shared example already covers.

| Shared example | Defined in | Key params |
|---|---|---|
| `"rendering Box"` | `box.rb` | `row_count:`, `header: true`, `footer: false` |
| `"rendering Blank Slate"` | `blank_slate.rb` | `heading:`, `icon: nil` |
| `"rendering Border Box List heading"` | `border_box_list_component.rb` | `text:`, `level: nil` |
| `"rendering an empty Border Box List"` | `border_box_list_component.rb` | `heading:`, `icon:`, `row_count:`, `header:` |
| `"a reorderable Border Box List"` | `border_box_list_component.rb` | `drag_type:` |

`"rendering an empty Border Box List"` already composes `"rendering Box"` +
`"rendering Blank Slate"` — use it instead of those two by hand. Don't hand-roll
DnD assertions when `"a reorderable Border Box List"` exists. When an assertion
block repeats across specs, extract a new shared example.

## Structure + semantics, not bare presence

Assert real elements inside their structural containers, not text that merely
appears somewhere:

```ruby
expect(rendered).to have_css(".Box-header") do |header|
  expect(header).to have_heading("Members", level: 3)
end
```

Approved matcher vocabulary:

- `have_heading(text, level: 4)`
- `have_button("Add")`, `have_button(accessible_name: "Edit list")`
- `have_link("Manage", href: "/members")`
- `have_octicon(:plus)` (e.g. blank-slate icon)
- `have_list` / `have_list_item`
- `have_css(".Box-row", text: …)` — prefer over bare `have_text` for row content
- `have_no_css(...)` / `have_no_text(...)` for negatives (e.g. proving a
  `with_title` slot *replaces* the `title:` kwarg)

## Non-tautological

A consumer spec must assert the consumer's **own** migrated markup — its data
attrs, drag type, locale keys, row content. If the spec would still pass while
rendering a totally different list, it proves nothing.

## Gotchas

- **Action-menu accessible name comes from a `tool-tip`,** not the button. Assert
  the element + tooltip, not `have_button`:
  ```ruby
  expect(rendered).to have_css("action-menu")
  expect(rendered).to have_css("tool-tip[data-type='label']", text: I18n.t(:label_actions))
  ```
- **Trailing `kw:` shorthand at end of line** triggers Ruby line-continuation —
  parenthesize chained `it_behaves_like(...)`.
- **Route-coupled components:** wrap the render in `with_request_url("…")`.
- Prefer **semantic locators** over `data-test-selector` (use the selector only
  when no stable semantic locator exists, per `CLAUDE.local.md` "## Spec Style").
- `click_on` over `click_link` / `click_link_or_button`.
- `Class.new` for throwaway registry values; `class_double` when storing a class.
- Control record counts explicitly (`create_list(:meeting, 0|2)`); `reload`
  cached associations. No `tap`, no `allow_any_instance_of`.
