# HEEx Control Flow

## Rule

Prefer HEEx `:if` and `:for` directives when the condition or iteration applies to a single element, component, or slot entry.

Use block control flow when it is genuinely clearer or when the branch needs to render multiple sibling nodes.

## Prefer Directives For Local Markup Decisions

Use `:if` and `:for` when they keep the template close to the markup being controlled.

Prefer:

```heex
<li :for={entry <- @entries}>{entry.label}</li>
```

Prefer:

```heex
<p :if={@error_message} class="text-sm text-rose-700">
  {@error_message}
</p>
```

Prefer:

```heex
<.badge :if={show_beta_badge?(@feature)} kind="info">
  Beta
</.badge>
```

Avoid:

```heex
<%= for entry <- @entries do %>
  <li>{entry.label}</li>
<% end %>
```

Avoid:

```heex
<%= if @error_message do %>
  <p class="text-sm text-rose-700">
    {@error_message}
  </p>
<% end %>
```

## Use Blocks When The Control Flow Owns The Structure

Block `if`, `case`, or `for` is still appropriate when:

- the branch renders multiple sibling nodes
- the template needs intermediate bindings before the markup
- the control flow is easier to read as a larger block than as repeated directives

Prefer:

```heex
<%= if @show_summary? do %>
  <h2>Summary</h2>
  <p>{@summary}</p>
<% end %>
```

Prefer:

```heex
<%= case @state do %>
  <% :loading -> %>
    <.loading_state />
  <% :empty -> %>
    <.empty_state />
  <% :loaded -> %>
    <.results_table rows={@rows} />
<% end %>
```

## Avoid Template Gymnastics

Do not contort the template just to force directive syntax everywhere.

If directive-heavy markup becomes harder to scan, extract a function component or keep the clearer block form.

## Review Questions

- Does this `:if` or `:for` apply to one element or component?
- Would a block make the structure clearer because multiple siblings are involved?
- Am I keeping the control flow next to the markup it controls?
- Is the template being bent around the rule instead of made clearer by it?
