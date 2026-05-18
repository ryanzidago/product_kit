# Composition

## Rule

Compose from existing primitives before creating new ones.

Prefer layering small, stable building blocks over cloning markup into every feature.

## Start With Shared Primitives

Before adding a new component, ask:

- does a project core component already solve this?
- does a shared UI primitive already exist?
- can this feature-specific wrapper be built from existing parts?

Creating a new wrapper is fine when it clarifies a real recurring pattern. Creating a new primitive for every screen is not.

## Keep Responsibilities Clean

The parent should usually own:

- layout
- spacing between siblings
- page-level visibility decisions
- higher-level composition

The child should usually own:

- its internal structure
- its semantic states
- its own accessible naming and affordances

For layout and spacing defaults, follow the broader [UI guide](../../../ui-ux/README.md).

## Avoid Duplicate Markup Families

If the same pattern appears in multiple places with minor tweaks, prefer:

- extending the existing component contract
- adding a variant
- adding a slot
- creating a narrow feature wrapper around the shared primitive

Do not fork the same design into many near-identical components without a boundary reason.

## Review Questions

- Am I composing from an existing primitive, or reimplementing one?
- Does this wrapper add real meaning, or only rename markup?
- Does the parent own layout while the child owns internals?
- If this pattern changes later, how many copies would need to move?
