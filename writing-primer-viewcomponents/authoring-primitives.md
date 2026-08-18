# Authoring a primitive — worked example

One complete primitive. It follows the recipes in `SKILL.md`: the house
`StringInquirer` enum idiom, slots that forward system arguments, slot-dependent
work in `before_render`, a real `render?`, one spelling per attribute, and a
boolean named for its effect.

Reference, don't copy verbatim — adapt names to your component.

## 1. The component (`.rb`)

```ruby
# frozen_string_literal: true

# (GPL copyright header here — copy from any sibling component file)

module OpenProject
  module Common
    # A bordered panel: a header (title + optional count + optional actions)
    # over caller-supplied rows, with an automatic empty state.
    class SummaryPanelComponent < ApplicationComponent
      include OpPrimer::ComponentHelpers
      include Primer::FetchOrFallbackHelper

      SCHEME_DEFAULT = :default
      SCHEME_OPTIONS = [SCHEME_DEFAULT, :condensed].freeze

      attr_reader :title, :scheme, :count

      # Repeated body rows. The caller owns the content; the panel owns the
      # list semantics only.
      #
      # @!parse
      #   # @param system_arguments [Hash] forwarded to the row list item.
      #   # @return [ViewComponent::Slot]
      #   def with_row(**system_arguments, &block); end
      renders_many :rows, ->(**system_arguments) do
        system_arguments[:tag] = :li
        system_arguments[:classes] = class_names(system_arguments[:classes], "op-summary-panel--row")
        Primer::BaseComponent.new(**system_arguments)
      end

      # Optional overflow menu. Returns a wrapper that owns the default kebab
      # trigger but delegates the full ActionMenu item API to the caller.
      #
      # @!parse
      #   # @param system_arguments [Hash] forwarded to Primer::Alpha::ActionMenu.
      #   # @return [ViewComponent::Slot]
      #   def with_menu(**system_arguments, &block); end
      renders_one :menu, ->(**system_arguments) do
        Menu.new(menu_id: "#{@panel_id}_menu", **system_arguments)
      end

      # Optional custom empty state; a generic default is injected when omitted.
      #
      # @!parse
      #   # @param title [String]
      #   # @param description [String, nil]
      #   # @param icon [Symbol, nil]
      #   # @return [ViewComponent::Slot]
      #   def with_empty_state(title:, description: nil, icon: nil); end
      renders_one :empty_state, ->(title:, description: nil, icon: nil) do
        EmptyState.new(title:, description:, icon:, announce_changes: @announce_changes)
      end

      # @param title [String] header title.
      # @param scheme [Symbol] <%= one_of(SummaryPanelComponent::SCHEME_OPTIONS) %>
      # @param count [Integer, Boolean, nil] nil/false hides the badge, true
      #   infers it from the rendered rows, an integer sets it explicitly.
      # @param announce_changes [Boolean] announce count / empty-state updates
      #   politely to assistive technology.
      # @param system_arguments [Hash] <%= link_to_system_arguments_docs %>
      def initialize(title:, scheme: SCHEME_DEFAULT, count: nil, announce_changes: false, **system_arguments)
        super()
        @title = title
        @scheme = ActiveSupport::StringInquirer.new(fetch_or_fallback(SCHEME_OPTIONS, scheme, SCHEME_DEFAULT).to_s)
        @count = count
        @announce_changes = announce_changes

        @system_arguments = system_arguments
        @panel_id = @system_arguments[:id] ||= self.class.generate_id
        @header_id = "#{@panel_id}_header"
        @system_arguments[:padding] = :condensed if condensed?
        @system_arguments[:classes] = class_names(
          @system_arguments[:classes],
          "op-summary-panel",
          "op-summary-panel_condensed" => condensed?
        )
      end

      # Slot-dependent work happens here, after `content` captures the slots.
      def before_render
        content
        @count = rows.size if @count == true
        configure_empty_state!
      end

      # Real presence gate — never `|| true`.
      def render? = rows.any? || empty_state? || menu?

      def condensed? = scheme.condensed?
      def render_count? = !@count.nil? && @count != false

      private

      def configure_empty_state!
        return if rows.any? || empty_state?

        with_empty_state(title: I18n.t(:label_nothing_display))
      end
    end
  end
end
```

Note the enum: `@scheme` is a `StringInquirer`, so `condensed?` reads as
`scheme.condensed?` and the CSS modifier comes from a `class_names` conditional.
The reader returns a `String`, not a `Symbol` — compare `scheme.condensed?` or
`scheme == "condensed"`, never `scheme == :condensed`.

The boolean is `announce_changes:` (its effect), not `interactive:`. The menu
has exactly one configuration path. `render?` gates on real slots.

## 2. The template (`.html.erb`)

Title text and count sit on the heading line; the menu is a **sibling**, never
inside the heading. Rows are a `<ul>` linked to the header via `aria-labelledby`.

```erb
<%= render(Primer::Beta::BorderBox.new(**@system_arguments)) do |box| %>
  <% box.with_header(id: @header_id) do %>
    <%= render(Primer::Beta::Truncate.new(tag: :h3, classes: "Box-title")) { title } %>
    <% if render_count? %>
      <%= render(Primer::Beta::Counter.new(count: @count)) %>
    <% end %>
    <%= menu if menu? %>
  <% end %>
  <% box.with_body(p: 0) do %>
    <% if rows.any? %>
      <%= content_tag(:ul, class: "op-summary-panel--rows", aria: { labelledby: @header_id }) do %>
        <% rows.each { |row| %><%= row %><% } %>
      <% end %>
    <% else %>
      <%= empty_state %>
    <% end %>
  <% end %>
<% end %>
```

## 3. Preview (`lookbook/previews/...`)

Subclass `ViewComponent::Preview`. Annotate the class (`@logical_path`,
`@display`) and each example (`@label`, `@param name [Type] select [...]` /
`text` / `toggle`). Cast toggles via a private helper
(`ActiveModel::Type::Boolean.new.cast`). Mirror the doc's variants.

```ruby
# @logical_path OpenProject/Common
class SummaryPanelComponentPreview < ViewComponent::Preview
  # @label Default
  def default
    render(OpenProject::Common::SummaryPanelComponent.new(title: "Open items", count: true)) do |panel|
      panel.with_menu { |menu| menu.with_item(label: "Configure") }
      panel.with_row { "First" }
      panel.with_row { "Second" }
    end
  end

  # @label Empty
  def empty
    render(OpenProject::Common::SummaryPanelComponent.new(title: "Attachments"))
  end
end
```

## 4. Docs (`lookbook/docs/components/summary-panel.md.erb`)

`.md.erb`, embeds previews with `<%= embed PreviewClass, :example %>`. Structure:
intro → Overview embed → Anatomy → Parameters table → Slots table → Variants →
Usage guidelines → Code structure. Document **only the params your component
owns** — do not list forwarded Primer args (e.g. `padding:`) as owned params;
that drift is exactly the bug in the real `border-box-list.md.erb`.

## YARD for slots: `@!parse`

Slots are defined by lambdas, so there is no real method to document. Precede
each `renders_one`/`renders_many` with a `# @!parse` block declaring a fake
`def with_<slot>(...)` — and keep its signature in sync with the lambda,
**including `&block`**. Use `<%= one_of(FOO_OPTIONS) %>` for enum params and
`<%= link_to_system_arguments_docs %>` for the `system_arguments` param.

## Keyword-argument ordering (when calling Primer slots)

Order kwargs by concern (from `CLAUDE.local.md`):

1. explicit slot params (in lambda order)
2. typography (`font_weight:`, `font_size:`)
3. layout / display (`display:`, `align_items:`, `justify_content:`, `flex:`)
4. spacing (`p:`, `m:`, …)
5. other system args (`color:`, `bg:`, `border:`, `data:`, `aria:`)
6. `classes:` **last**

## `merge_aria` / `merge_data` (when you must merge ARIA/data)

Include `Primer::AttributesHelper` **only in the component that actually merges**
(not blanket on every component). Semantics from
`app/lib/primer/attributes_helper.rb`:

- **Non-plural keys: later hash wins.** Pass your defaults **first**, the live
  caller hash **second**, so caller values override defaults:
  `merged[:aria] = merge_aria({ aria: { label: default } }, merged)`.
- **Plural keys are space-joined**, not overwritten — aria: `describedby`,
  `labelledby`; data: `target`, `targets`, `action`.
- **It mutates its arguments**: it `delete`s `:aria`/`:data` and the consumed
  `aria-*`/`data-*` flat keys off the hashes you pass. `deep_dup` a caller hash
  before merging if you still need the original.

## Wrapper / facade components

If you wrap a Primer component (e.g. a `Menu` over `Primer::Alpha::ActionMenu`)
and want to expose its API, **don't hand-maintain a hardcoded `delegate`
allowlist** — it silently drops the rest of the wrapped API and rots on every
Primer upgrade. Either delegate broadly (`method_missing` /
`respond_to_missing?`) or deliberately document a narrow facade and say why.
Inject your default (e.g. the kebab trigger) in `before_render` only when the
caller hasn't already configured it (track with a flag).
