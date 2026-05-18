# Documentation

## Rule

Document stable intent, boundaries, and invariants.

This guide applies across the workspace, including:

- business logic in contexts
- domain and workflow modules
- database-facing modules
- Phoenix LiveViews and components
- background jobs and integration modules
- shared test helpers and support code

Prefer documentation that survives refactors:

- why this code exists
- why this boundary matters
- when to use this module or function
- when not to use it
- what invariants or side effects callers must respect

Avoid documentation that mostly repeats the current implementation shape. Code, types, tests, and `attr` declarations already show a lot of the "what". Documentation should earn its keep by explaining the "why" and the "so what".

## Module-Level Documentation

Use `@moduledoc` for modules that have a real responsibility, boundary, or caller-facing purpose.

This includes:

- context modules
- business workflow modules
- query or persistence modules when they expose a real boundary
- LiveViews, components, and helper modules
- jobs, adapters, and integration modules
- shared test support modules

Good module docs usually answer:

- why this module exists
- what responsibility it owns
- when another module should call it
- when another module should not call it
- what important boundary or invariant it protects

For feature modules, prefer documenting the role the module plays in the system over the list of functions it contains.

This matters especially for contexts and domain modules. Their docs should explain the business capability they own and the boundary they are meant to protect, not just the operations they happen to expose today.

Prefer:

```elixir
@moduledoc """
Coordinates payroll approval runs.

Use this module when a workflow needs to start, advance, or cancel an approval run.
Do not use it for reporting queries or page-specific formatting; those belong in reporting
modules and presentation code.

This boundary matters because approval runs enforce the sequencing and audit rules that must
stay consistent regardless of whether the caller is a LiveView, job, or API endpoint.
"""
```

Avoid:

```elixir
@moduledoc """
Contains functions for creating, updating, listing, and deleting approval runs.
"""
```

### What To Include

- the business or UI concept the module represents
- the main reason the module was introduced
- the correct entry points for callers
- important exclusions so the boundary stays clear
- cross-cutting constraints that are easy to violate accidentally

### What To Avoid

- narrating the file structure
- listing every public function
- describing step-by-step implementation mechanics
- copying type information that is already obvious from specs

### LiveViews Need Purpose Docs Too

Important LiveViews should use `@moduledoc` to explain the page-level workflow and boundary.

Good LiveView docs usually answer:

- what user workflow or page responsibility this LiveView owns
- what state is URL-backed or otherwise important to restore
- which contexts or helper modules hold the real business logic
- what should stay out of the LiveView

Do not use `@moduledoc` to enumerate assigns, callback names, or render branches. That information changes too often and is already visible in the code.

### Contexts And Domain Modules Need Boundary Docs Too

Important context and business-logic modules should document:

- the business capability they own
- the invariants or policy decisions they enforce
- which callers should come through this boundary
- which concerns belong somewhere else, such as page formatting or raw reporting queries

If a module is where the real rules live, its docs should make that obvious.

### When `@moduledoc false` Is Better

Use `@moduledoc false` when the module is intentionally internal and its purpose is already obvious from a narrow local context.

Good candidates:

- tiny private support modules nested inside a file
- generated code
- low-level glue modules with no real caller-facing boundary

Do not hide a meaningful boundary behind `@moduledoc false` just because the module feels "internal". If another engineer has to decide whether to use it, the module probably needs documentation.

## Function-Level Documentation

Use `@doc` for public functions whose contract is not fully obvious from the name, types, and calling context.

Good function docs usually answer:

- what contract the caller relies on
- what important side effects happen
- what shape of failure to expect
- what assumptions or invariants the caller must satisfy
- why this function exists instead of using a more obvious alternative

Prefer:

```elixir
@doc """
Approves a run if the actor still has permission at execution time.

Returns `{:error, :stale_permission}` when the run was valid when rendered but the actor no
longer has access when the action executes.
"""
```

Avoid:

```elixir
@doc """
Approves a run.
"""
```

### Document Semantics, Not Spelling

`@doc` should explain meaning that the function name does not already carry.

Useful topics:

- idempotency
- retries
- transaction boundaries
- authorization expectations
- whether the function is safe for background jobs
- whether it is intended for UI orchestration or domain use

Skip `@doc` or use `@doc false` when:

- the function is private
- the function is public only to satisfy a behaviour or macro callback and extra prose adds nothing
- the function is a thin, obvious wrapper with no additional semantics

Do not write docs that become false whenever internal steps change.

## Comments

Comments are for context the code cannot carry well on its own.

Use comments to explain:

- invariants
- non-obvious tradeoffs
- external system quirks
- ordering requirements
- compatibility constraints
- why a surprising approach is intentional

Prefer:

```elixir
# Keep export columns in this order because downstream payroll imports match by position.
@export_columns [:employee_id, :period_start, :period_end, :gross_pay]
```

Avoid:

```elixir
# Define the export columns.
@export_columns [:employee_id, :period_start, :period_end, :gross_pay]
```

### Prefer Better Names First

Before adding a comment, ask whether a better function name, variable name, helper extraction, type, or `attr` declaration would make the code self-explanatory.

Comment only after the code shape is already as clear as it should reasonably be.

### Comment Lifetimes Matter

Delete or rewrite comments when the reason changes. Stale comments are worse than missing comments because they create false confidence.

If a comment captures a temporary workaround, make that explicit and tie it to a concrete condition, issue, or dependency.

## Test Documentation

Tests are documentation for behaviour. Treat test names as the primary documentation surface.

Prefer test names that explain:

- the user-visible behaviour
- the invariant being protected
- the regression being prevented

Prefer:

```elixir
test "restores the selected filter from the URL after refresh" do
```

Avoid:

```elixir
test "handle_params assigns filters" do
```

### What To Document In Tests

Document the scenario, not the mechanics.

Good places for documentation:

- test names
- `describe` blocks that frame a workflow or boundary
- module docs on shared test helpers and support modules
- short comments that explain a real-world failure or non-obvious fixture choice

Useful test comments usually explain one of:

- why a fixture must look unrealistic or unusually specific
- which production bug or regression the test protects against
- why an assertion matters even if the markup later changes

Avoid comments that narrate the arrange/act/assert steps line by line.

### Test Support Modules Need Real Docs

Shared test helpers, factories with non-obvious defaults, and support modules should use `@moduledoc` and `@doc` like production code when they expose reusable contracts.

The point is the same: help the next engineer understand when to reach for the helper and what assumptions it bakes in.

## HEEx `attr` Documentation

For function components, `attr` and `slot` declarations are part of the documentation.

They should make the component contract obvious:

- what inputs exist
- which are required
- which have defaults
- which values are accepted

Use `doc:` on `attr` when the semantic meaning is not obvious from the name and type alone.

Prefer:

```elixir
attr :rest, :global, doc: "Extra HTML attributes passed to the outer button"
attr :target, :string, doc: "DOM id that receives the emitted JS event"
```

Avoid:

```elixir
attr :class, :string, doc: "The class"
attr :label, :string, doc: "The label"
```

### `attr` Does Not Replace All Prose

`attr` documents the component API. It does not explain:

- why the component exists
- when to use this component instead of another one
- accessibility or behaviour constraints that span multiple attrs

That higher-level context still belongs in `@moduledoc` when the component is shared or important.

### Keep `attr` Focused On Components

Do not use `attr` to document a LiveView's own `render/1` assigns or non-component helpers. That is covered by the component attrs guide and the LiveView lifecycle itself.

See [elixir/phoenix/components/attrs.md](../elixir/phoenix/components/attrs.md) for the detailed `attr` rules.

## Review Questions

- Does this documentation explain why the code matters, or only what the current implementation does?
- If the implementation were refactored tomorrow, would most of this documentation still be true?
- Would a new engineer learn when to use this module or function, and when not to?
- Are comments carrying information that the code could express more clearly by itself?
- Do the test names document behaviour, not callbacks or assigns?
- For components, do `attr` and `doc:` explain the caller-facing contract without replacing higher-level module docs?
