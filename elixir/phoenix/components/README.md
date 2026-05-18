# Writing Components

This folder contains the default workspace conventions for writing Phoenix components.

Each guide is self-contained on purpose:

- humans can read only the topic they need
- LLMs can retrieve a narrow file without dragging in unrelated guidance
- review comments can link to a specific convention instead of a long mixed document

## Guide Index

- [boundaries.md](boundaries.md) - Choose function components, LiveComponents, and LiveViews intentionally.
- [api-design.md](api-design.md) - Design component APIs around meaning, not implementation details.
- [attrs.md](attrs.md) - Use `attr` for real component contracts, including page-local components inside LiveViews.
- [../heex.md](../heex.md) - Prefer HEEx `:if` and `:for` directives when they keep control flow local to the markup.
- [composition.md](composition.md) - Compose from existing primitives before creating new ones.
- [state-and-events.md](state-and-events.md) - Keep state, events, and side effects in the right layer.
- [accessibility.md](accessibility.md) - Treat accessibility as part of the component contract.

## Core Position

The broad shape across all guides is:

- function components by default
- LiveComponents only when local state or event targeting justifies them
- LiveViews own page orchestration
- contexts own business rules
- component APIs express semantic intent
- accessibility is part of the contract, not polish

Read the topic guide that matches the decision you are making. If multiple guides apply, prefer the lighter abstraction and the stricter boundary.
