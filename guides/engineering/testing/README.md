# Testing Guides

This folder contains the default workspace conventions for testing topics that are broader than one framework layer.

Each guide is self-contained on purpose:

- humans can read only the topic they need
- LLMs can retrieve a narrow file without dragging in unrelated guidance
- review comments can link to a specific convention instead of a long mixed document

## Guide Index

- [abstract-vs-real-world.md](abstract-vs-real-world.md) - Use both minimal contract tests and realistic domain-shaped examples when behavior matters.
- [global-state-isolation.md](global-state-isolation.md) - Keep tests isolated from shared global process and application state.
- [test-suite-shape.md](test-suite-shape.md) - Shape the suite around confidence and risk, with strong emphasis on integration and LiveView tests over browser-heavy coverage.

## Core Position

The broad shape across all guides is:

- prove the rule clearly
- also prove it against realistic data where it matters
- keep tests isolated from shared mutable state
- bias toward the test layer that best matches the risk
- prefer test shapes that make failures easy to diagnose

Read the narrowest guide that matches the testing decision you are making.
