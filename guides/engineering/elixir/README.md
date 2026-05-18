# Elixir Guides

This folder contains the default workspace conventions for Elixir application code outside narrower framework-specific topics.

Each guide is self-contained on purpose:

- humans can read only the topic they need
- LLMs can retrieve a narrow file without dragging in unrelated guidance
- review comments can link to a specific convention instead of a long mixed document

## Guide Index

- [style-guide.md](style-guide.md) - Core Elixir coding conventions for naming, control flow, map access, specs, and common defaults.
- [function-purity.md](function-purity.md) - Keep side effects at explicit boundaries and push decision-making into pure functions.
- [error-handling.md](error-handling.md) - Keep failure contracts explicit and translate errors at the boundary that owns them.
- [background-jobs.md](background-jobs.md) - Use jobs intentionally and keep workers thin, idempotent, and operationally clear.
- [external-system-integrations.md](external-system-integrations.md) - Structure vendor integrations behind app-owned boundaries with testable adapters.
- [iex-examples.md](iex-examples.md) - Write copy-pasteable IEx examples that are easy to run and reason about.
- [localization.md](localization.md) - Separate time semantics, translated copy, and locale-aware formatting.
- [observability.md](observability.md) - Emit logs, telemetry, and error signals at workflow boundaries instead of scattering instrumentation through helpers.
- [parameter-extraction.md](parameter-extraction.md) - Extract required params explicitly and keep missing-input handling clear.
- [time-handling.md](time-handling.md) - Model dates, times, timezones, and serialization intentionally.
- [ecto/README.md](ecto/README.md) - Entry point for Ecto-specific guides.
- [phoenix/README.md](phoenix/README.md) - Entry point for Phoenix-specific guides.

## Core Position

The broad shape across all guides is:

- explicit boundaries over magical convenience
- pure decision logic separated from effectful boundaries
- small, readable functions over clever compression
- domain-meaningful error contracts
- observability at workflow boundaries over helper-level noise
- time and params modeled intentionally
- thin edges around jobs, databases, and integrations

Read the narrowest guide that matches the decision you are making. If multiple guides apply, prefer the stricter boundary.
