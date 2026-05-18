# Attr Declarations

## Rule

Use `attr` for function components that have a real caller-facing contract.

That includes:

- shared component modules
- private or page-local function components extracted inside a LiveView

Do not use `attr` for a LiveView's `render/1`, callbacks, or plain helper functions.

## Use `attr` When The Function Is A Component

If a function is rendered as `<.name ...>`, declare its inputs with `attr` and `slot`.

This stays true even when the component is:

- private
- used by one LiveView only
- defined in the same module as the LiveView

Once markup has been extracted into a named component, its contract should be explicit.

## In LiveViews, Separate Page Assigns From Component Attrs

LiveView assigns such as `@current_user`, `@filters`, `@page`, or `@form` are owned by `mount/3`, `handle_params/3`, and event handlers.

`attr` is not how you declare that page-level state. Use normal assigns for that layer.

Use `attr` only for extracted rendering units inside the LiveView.

Prefer:

```elixir
def render(assigns) do
  ~H"""
  <.summary_card title="Errors" count={@error_count} />
  """
end

attr :title, :string, required: true
attr :count, :integer, required: true

defp summary_card(assigns) do
  ~H"""
  <section>
    <h2>{@title}</h2>
    <p>{@count}</p>
  </section>
  """
end
```

Avoid using `attr` to document the LiveView's own `render/1` assigns. That contract belongs in the LiveView lifecycle, not in component declarations.

## Do Not Use `attr` For

- `render/1` in a LiveView
- `mount/3`, `handle_params/3`, `handle_event/3`, or `handle_info/2`
- plain helpers that return classes, labels, booleans, maps, or transformed data
- helper modules such as `...Live.Helpers` that shape page state rather than render markup
- opaque escape hatches such as a single `attr :options, :map` when the real API is known

## Prefer Small, Explicit Contracts

- declare each meaningful input explicitly
- mark attrs `required: true` only when the component cannot make sense without them
- use defaults for optional behaviour
- use `slot` for meaningful content areas instead of ad hoc content assigns
- use `:global` only when you intentionally support caller-supplied HTML attributes

Do not skip `attr` just because a component is local or private. Do not add `attr` just because a function accepts `assigns`.

## `attr` Versus `assign_new`

Use `attr` to declare the public contract. Use `assign_new` to fill in a missing value at runtime.

They solve different problems:

- `attr` says what callers may pass
- `default:` says the component has a fixed caller-visible fallback
- `assign_new` computes or derives a value only when the caller did not provide one

Prefer `default:` on `attr` when the fallback is simple, stable, and part of the API.

Prefer:

```elixir
attr :size, :atom, default: :medium
attr :disabled, :boolean, default: false
```

Prefer `assign_new` when the fallback depends on other assigns, must be generated, or should stay as implementation detail.

Prefer:

```elixir
attr :id, :string, default: nil
attr :kind, :atom, required: true

def flash(assigns) do
  assigns = assign_new(assigns, :id, fn -> "flash-#{assigns.kind}" end)

  ...
end
```

Prefer:

```elixir
attr :field, Phoenix.HTML.FormField, default: nil
attr :name, :string, default: nil

def input(%{field: %Phoenix.HTML.FormField{} = field} = assigns) do
  assigns
  |> assign_new(:name, fn -> field.name end)
  |> input()
end
```

Do not use `assign_new` as a substitute for declaring real attrs. If callers are expected to pass a value, declare it with `attr` first.

If a value is purely internal and callers should not override it, derive it with `assign/3` or a local variable instead of exposing it as an attr.

## When To Use `default:`

Use `default:` when omitting the attr should still produce valid, unsurprising behaviour and the fallback is the same for every caller.

Good candidates:

- booleans such as `disabled: false`
- styling variants with a clear normal case
- empty lists or maps when empty is a real, supported state
- optional labels, classes, and HTML attributes when omission is normal

Use `required: true` instead when the component would be ambiguous, misleading, or broken without the value.

Good candidates:

- semantic content such as `title`, `label`, `status`, or `rows`
- callbacks the component cannot execute without
- IDs that must be stable and caller-chosen
- values where any default would hide a bug at the call site

Avoid `default: nil` unless `nil` is a real supported state that the component handles intentionally.

Avoid redundant declarations such as `required: false` when a default already makes the attr optional.

## When To Use `doc:`

Use `doc:` when it helps a caller understand semantics, constraints, or behaviour that are not obvious from the attr name and type alone.

`doc:` is especially useful for:

- shared component libraries
- callbacks and function attrs
- attrs with allowed values or behavioural consequences
- `:global` attrs and custom include lists
- IDs, targets, and event-related attrs
- attrs typed as `:any`

Skip `doc:` when the component is very local and the attr is truly obvious.

Prefer concise docs that explain meaning, not the spelling of the attr name.

Prefer:

- `doc: "Stable id base used for aria relationships"`
- `doc: "Called when the filter text changes"`
- `doc: "Extra HTML attributes passed to the wrapper"`

Avoid:

- `doc: "The class of the class"`
- `doc: "The label"`
- `doc: "The id of the component"`

## Do Not Extract Just To Use `attr`

Keep markup inline in `render/1` when it is truly one-off and has no stable API.

Extract into a function component when at least one of these is true:

- the markup repeats on the page
- the UI has a clear name or concept
- a smaller rendering unit improves readability
- the inputs are worth validating as a contract

Once extracted into a function component, add `attr` immediately.

## Review Questions

- Is this a real function component, or just a helper?
- Are these inputs part of page orchestration or a component API?
- Should this fallback be an `attr default`, an `assign_new`, or an internal `assign`?
- Would a missing value be safe and obvious, or should it be required?
- Does `doc:` explain semantics, or only restate the attr name?
- Would a caller understand the contract from the `attr` declarations alone?
- Are `required` and `:global` precise, or being used as shortcuts?
- Should this stay inline instead of becoming a component?
