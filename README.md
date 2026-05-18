# Shared Guides

This folder contains reusable guidance that applies across projects that adopt this guide set.

Use the top-level files for cross-cutting topics. Use the subfolders for topic-specific or stack-specific guidance.

## Guide Map

- [api-design.md](api-design.md) - How to shape public and internal APIs around honest async boundaries, idempotent writes, durable workflow resources, and explicit operational contracts.
- [business-logic-computation.md](business-logic-computation.md) - How to structure calculation-heavy domain logic such as holiday, hours, and scoring rules.
- [transactional-vs-analytical-databases.md](transactional-vs-analytical-databases.md) - How to separate operational source-of-truth state from derived analytical reads and data flows.
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
- [architecture.md](architecture.md) - Architecture and system-shape guidance.
- [code-review.md](code-review.md) - How to review for concrete bugs, regressions, missing safeguards, and test gaps.
- [communication.md](communication.md) - Communication and collaboration conventions.
- [data-privacy-and-compliance.md](data-privacy-and-compliance.md) - How to minimize sensitive data, keep it out of logs and exports, and make retention and deletion explicit.
- [data-model-and-schema-design.md](data-model-and-schema-design.md) - How to model durable domain concepts in the schema and make invalid states hard to represent.
- [design-plans.md](design-plans.md) - How to draft decision-first design plans from the patterns used in this workspace's artifacts.
- [documentation.md](documentation.md) - How to document modules, functions, comments, tests, and HEEx component attrs.
- [implementation-plans.md](implementation-plans.md) - How to draft executable implementation plans that turn an approved design into ordered, verifiable tasks.
- [observability.md](observability.md) - How to make important features diagnosable with the right usage, failure, latency, and correlation signals.
- [post-release-verification.md](post-release-verification.md) - How to verify important changes in production after deploy by watching logs, errors, latency, and feature-specific outcomes.
- [pr-splitting.md](pr-splitting.md) - How to split stacked pull requests by meaningful behavior slices instead of technical layers.
- [reliability-and-resilience.md](reliability-and-resilience.md) - How to make important workflows safe under retries, partial failure, timeouts, and interrupted execution.
- [retrospective.md](retrospective.md) - How to write useful retrospectives.
- [security-and-authorization.md](security-and-authorization.md) - How to enforce trust boundaries, tenant scoping, and sensitive action protection at the right layer.
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

Workspace-specific operational runbooks belong in the relevant workdir under `guides/workspace/`, not here.
