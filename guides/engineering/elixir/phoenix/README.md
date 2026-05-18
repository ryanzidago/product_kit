# Phoenix Guides

This folder contains the default workspace conventions for Phoenix presentation code.

Each guide is self-contained on purpose:

- humans can read only the topic they need
- LLMs can retrieve a narrow file without dragging in unrelated guidance
- review comments can link to a specific convention instead of a long mixed document

## Guide Index

- [heex.md](heex.md) - Prefer HEEx directives when they keep control flow local to the markup.
- [components/README.md](components/README.md) - Entry point for Phoenix component guidance.
- [liveviews/README.md](liveviews/README.md) - Entry point for Phoenix LiveView guidance.

## Core Position

The broad shape across all guides is:

- HEEx stays readable and local
- components express reusable UI contracts
- LiveViews orchestrate pages and interactions
- business logic stays outside the presentation layer
- accessibility is part of the default contract

Read the narrowest guide that matches the decision you are making. If multiple guides apply, prefer the lighter abstraction and the stricter boundary.
