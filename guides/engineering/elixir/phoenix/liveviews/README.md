# Writing LiveViews

This folder contains the default workspace conventions for writing Phoenix LiveViews.

Each guide is self-contained on purpose:

- humans can read only the topic they need
- LLMs can retrieve a narrow file without dragging in unrelated guidance
- review comments can link to a specific convention instead of a long mixed document

## Guide Index

- [routing.md](routing.md) - Keep LiveView route modules aligned with the newer Phoenix generator pattern.
- [navigation.md](navigation.md) - Use links for navigation and keep browser behaviour intact.
- [url-state.md](url-state.md) - Use the URL as the source of truth where possible.
- [../heex.md](../heex.md) - Prefer HEEx `:if` and `:for` directives when they keep control flow local to the markup.
- [forms-and-recovery.md](forms-and-recovery.md) - Design forms so progress survives refresh and reconnect.
- [orchestration.md](orchestration.md) - Keep LiveViews as orchestration layers.
- [page-states.md](page-states.md) - Handle loading, empty, error, and success states consistently.
- [streams.md](streams.md) - Use streams when they simplify long-lived lists, not by default.
- [modals.md](modals.md) - Choose patch-driven or local-state modals intentionally.
- [validation.md](validation.md) - Validate on change, blur, or submit with clear intent.
- [authorization.md](authorization.md) - Keep authorization as a real boundary, not just a UI concern.
- [testing.md](testing.md) - What important LiveViews should prove in tests.
- [javascript-boundaries.md](javascript-boundaries.md) - Use LiveView JS by default and hooks only when needed.
- [performance.md](performance.md) - Avoid oversized sockets, heavy rerenders, and noisy events.
- [lifecycle.md](lifecycle.md) - Keep `mount/3`, `handle_params/3`, `handle_event/3`, and `handle_info/2` sharply scoped.
- [redirects-and-flash.md](redirects-and-flash.md) - Choose `patch`, `navigate`, `push_patch`, `push_navigate`, and flash intentionally.
- [accessibility.md](accessibility.md) - Preserve semantics, focus, and keyboard access in dynamic UIs.
- [../components/attrs.md](../components/attrs.md) - Use `attr` for page-local function components, not page-level LiveView assigns.

## Core Position

The broad shape across all guides is:

- links for destinations
- buttons for actions
- URL-backed state where possible
- forms built for recovery
- business logic in contexts
- LiveViews as orchestration layers

Read the topic guide that matches the decision you are making. If multiple guides apply, prefer the stricter boundary.
