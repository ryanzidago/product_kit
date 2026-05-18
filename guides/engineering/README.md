# Engineering Guides

This folder contains reusable engineering guidance that applies across projects that adopt this guide set.

Use the category folders for cross-cutting topics. Use the stack-specific folders for implementation guidance.

## Guide Map

- [planning/](planning/) - Design plans, implementation plans, PR splitting, and retrospectives.
- [architecture/](architecture/) - System shape, API design, data modeling, security, privacy, reliability, and business computation.
- [delivery/](delivery/) - Code review, communication, documentation, observability, and post-release verification.
- [elixir/](elixir/) - Elixir application guidance, including core style, Ecto, and Phoenix LiveView guides.
- [elixir/function-purity.md](elixir/function-purity.md) - How to keep side effects at explicit boundaries and move decision logic into pure functions.
- [elixir/ecto/README.md](elixir/ecto/README.md) - Entry point for Ecto-specific guides.
- [elixir/observability.md](elixir/observability.md) - Where logs, telemetry, and error reporting belong in Elixir workflows, jobs, and integrations.
- [elixir/phoenix/components/README.md](elixir/phoenix/components/README.md) - Entry point for Phoenix component guidance.
- [elixir/phoenix/components/attrs.md](elixir/phoenix/components/attrs.md) - When to use `attr` declarations in components and LiveView-local function components.
- [elixir/phoenix/heex.md](elixir/phoenix/heex.md) - Prefer HEEx `:if` and `:for` directives when they keep control flow local to the markup.
- [elixir/phoenix/liveviews/README.md](elixir/phoenix/liveviews/README.md) - Entry point for Phoenix LiveView-specific guides.
- [testing/](testing/) - Testing guidance that is not tied to a single framework layer.
- [testing/global-state-isolation.md](testing/global-state-isolation.md) - How to avoid shared global state in tests.
- [testing/test-suite-shape.md](testing/test-suite-shape.md) - How to balance small tests, integration tests, LiveView tests, and selective browser end-to-end coverage.
- [ui-ux/README.md](ui-ux/README.md) - Entry point for reusable UI and UX guidance that is broader than Phoenix LiveView specifics.
- [ui-ux/performance-tradeoffs.md](ui-ux/performance-tradeoffs.md) - How to choose UI defaults, entry points, and progressive disclosure patterns that reduce system cost when the UX tradeoff is small.

## Elixir Layout

```text
elixir/
  style-guide.md
  function-purity.md
  error-handling.md
  background-jobs.md
  external-system-integrations.md
  iex-examples.md
  localization.md
  observability.md
  parameter-extraction.md
  time-handling.md
  ecto/
    README.md
  phoenix/
    heex.md
    components/
      README.md
    liveviews/
      README.md
```

Read the narrowest guide that matches the decision you are making.

Workspace-specific operational runbooks belong in the relevant project, not here.
