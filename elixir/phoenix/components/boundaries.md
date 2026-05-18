# Boundaries

## Rule

Choose the lightest Phoenix abstraction that matches the behaviour you need.

Default to function components for reusable rendering. Reach for LiveComponents only when they earn their cost.

## Prefer Function Components

Use function components when you are:

- rendering reusable markup
- standardising a visual pattern
- wrapping a shared form control or layout primitive
- composing other components together
- deriving display-only values from assigns

Use your project's component macro for this layer, for example `BackendWeb, :component` or `AgenticOsWeb, :component`.

## Use LiveComponents When They Add Real Value

Use a LiveComponent when you need one or more of these:

- local event targeting such as `phx-target={@myself}`
- local UI state that should not become page-level state
- lifecycle isolation that would be awkward in the parent LiveView
- a reusable interactive unit that genuinely benefits from its own callbacks

Do not use LiveComponents just because a piece of UI is large.

## Keep These Out Of Components

Whether the component is functional or stateful, do not let it quietly become the page:

- page-level data loading
- URL state ownership
- navigation orchestration
- real authorization decisions
- business rules or persistence

Those boundaries belong with the LiveView and the relevant context.

## Review Questions

- Could this stay a function component?
- Is the LiveComponent solving real state or event-targeting needs, or only size?
- Does this component own logic that should live in the parent LiveView or a context?
- If this component disappeared, would the page-level flow still make architectural sense?
