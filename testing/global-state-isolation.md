# Global State Isolation In Tests

## Rule

Tests should not rely on mutating shared global state to control behavior.

Avoid patterns like:

- `Application.put_env/3`
- `Application.put_env/4`
- `Application.put_all_env/1`
- `Application.put_all_env/2`
- `Application.delete_env/2`
- `Application.delete_env/3`
- `System.put_env/1`
- `System.put_env/2`
- `System.delete_env/1`

This applies even when the test module is synchronous. Async tests make the race condition obvious, but sync tests still create hidden coupling, harder-to-reason-about setup, and order-sensitive failures.

## Why

Global state is shared across the VM or the OS process.

When tests mutate it, they stop being fully owned by their setup. That makes failures harder to diagnose and makes it too easy for one test to change the behavior of another test, a helper, or a background process.

## Prefer

Prefer one of these patterns instead:

- pass the dependency or behavior in as a function argument
- inject a module dependency and swap it with a test double through ordinary parameters or assigns
- store test-controlled state in the test process, a supervised Agent, or another test-owned process
- define stable test configuration once in `config/test.exs` instead of rewriting app env inside individual tests
- use async-safe mocking or stubbing patterns when the codebase already has them

## Good Patterns

Prefer:

```elixir
def list_reports(account_id, catalog \\ ReportCatalog) do
  catalog.list_reports(account_id)
end
```

Prefer:

```elixir
socket = assign(socket, :schema_catalog, schema_catalog)
```

Prefer:

```elixir
Agent.start_link(fn -> %{response: {:ok, []}} end)
```

Avoid:

```elixir
Application.put_env(:my_app, :schema_catalog, MyTestCatalog)
```

Avoid:

```elixir
System.put_env("FEATURE_FLAG", "true")
```

## Acceptable Exceptions

If a test must change global state because the code cannot yet be injected cleanly:

1. treat it as a temporary exception, not the default pattern
2. keep the affected test module `async: false`
3. restore the original value in `on_exit`
4. keep the scope as small as possible
5. prefer creating follow-up work to remove the global mutation

Do not introduce new global-state test seams when a local seam can be added instead.

## Review Questions

- Can this test control behavior without rewriting application or system env?
- Is the dependency owned by the test, or shared by the VM?
- Would this test still be correct if another test ran at the same time?
- Should this seam be moved to ordinary dependency injection instead?
