# UI And UX Guides

This folder contains reusable UI guidance that is broader than one framework or component layer.

Each guide is self-contained on purpose:

- humans can read only the topic they need
- LLMs can retrieve a narrow file without dragging in unrelated guidance
- review comments can link to a specific convention instead of a long mixed document

## Guide Index

- [performance-tradeoffs.md](performance-tradeoffs.md) - How to choose UI defaults, entry points, and progressive disclosure patterns that reduce system cost when the UX tradeoff is small.

## Core Guide

## Core Rules

### 1. Parent Owns Layout, Child Owns Internals

Prefer higher-level layout and spacing utilities on the parent container before reaching for per-element margin and padding classes.

Use the parent for `flex`, `grid`, `gap-*`, `space-*`, `items-*`, `justify-*`, and responsive layout shifts. Use the child for its own internal padding, text flow, and local structure.

### 2. Prefer `gap-*` Over `space-*` When Possible

Use `gap-*` by default for stacks, grids, wrapped rows, and responsive layouts. Use `space-*` for simple one-dimensional stacks when it expresses the layout clearly.

`gap-*` tends to stay more predictable as layouts become more complex.

### 3. Use a Small, Deliberate Spacing Scale

Favor a narrow shared spacing scale such as `2`, `4`, `6`, and `8` rather than many near-duplicate values.

Consistent spacing creates rhythm faster than case-by-case tuning.

### 4. Avoid One-Off Nudges

Treat negative margins, arbitrary spacing values, and tiny corrective utilities as exceptions.

If a layout needs repeated nudges, the layout structure is usually wrong.

### 5. Responsive Changes Should Be Structural

Prefer changing the parent layout at breakpoints instead of patching children with many breakpoint-specific spacing overrides.

Switch from stack to row, or from one column to two columns, by changing the container.

### 6. Handle Flex Overflow Intentionally

In flex layouts, decide which elements may grow, shrink, wrap, or truncate.

Use utilities like `min-w-0`, `shrink-0`, `truncate`, and wrapping classes intentionally rather than accepting default overflow behavior.

### 7. Use Semantic Width Constraints

Prefer clear width constraints such as `w-full`, `max-w-prose`, `max-w-screen-lg`, `self-start`, and `shrink-0` over ad hoc width tuning on children.

### 8. Ship Complete Interaction States

Interactive UI should define relevant `hover`, `focus-visible`, `disabled`, selected, and loading states together.

State consistency is part of UI quality, not polish to add later.

### 9. Prefer Tokens Over Arbitrary Values

Use shared spacing, color, radius, border, and shadow tokens before reaching for arbitrary values.

Arbitrary values are acceptable when they are intentional and justified, not when they are faster than choosing the right token.

### 10. Design for Empty, Loading, and Error States

UI components and pages should account for loading, empty, error, and success states as part of the component contract.

The default or happy path is not the full design.

## Prefer

Use the parent to define layout and rhythm:

```heex
<div class="flex flex-col gap-4">
  <.header />
  <.filters />
  <.results />
</div>
```

```heex
<div class="grid gap-6 md:grid-cols-2">
  <.stat_card />
  <.stat_card />
</div>
```

```heex
<div class="flex flex-col gap-4 md:flex-row md:items-start">
  <aside class="shrink-0 md:w-64">
    <.filters />
  </aside>

  <main class="min-w-0 flex-1">
    <.results />
  </main>
</div>
```

## Avoid

Do not build layout by stacking child-specific spacing classes when a parent rule would express the structure more clearly:

```heex
<div>
  <.header class="mb-4" />
  <.filters class="mb-4 pt-2 md:mr-6" />
  <.results class="mt-3 md:ml-6" />
</div>
```

Do not rely on repeated nudges and arbitrary values to rescue a weak layout:

```heex
<div class="flex">
  <aside class="mr-[22px] mt-px w-[255px]">
    <.filters />
  </aside>

  <main class="ml-[13px] w-[calc(100%-268px)]">
    <.results />
  </main>
</div>
```

## Exceptions

- Use padding when a component needs internal space inside its own boundary.
- Use margin when expressing a real external relationship that the parent cannot own cleanly.
- Use arbitrary values when the design requirement is specific and the shared scale cannot express it well.
- Use one-off spacing utilities when the exception is intentional and rare, not when they are compensating for missing layout structure.

## Review Questions

- Can this layout responsibility move to the parent container?
- Should this relationship be expressed with `gap-*` instead of repeated child margins?
- Is this spacing value part of the shared scale, or a one-off nudge?
- Are breakpoint changes happening at the container level, or scattered across children?
- Will this flex row behave correctly when content gets long?
- Are width constraints explicit and semantic?
- Are all important interactive states defined?
- Does this UI still make sense in loading, empty, and error states?
