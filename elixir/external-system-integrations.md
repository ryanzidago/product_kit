# External System Integrations

## Rule

Model every external system behind a behaviour-backed facade and a separate implementation module.

Use this shape by default:

- `MyApp.Integrations.ExampleAPI` - public facade, callbacks, shared types
- `MyApp.Integrations.ExampleAPI.Impl` - production adapter
- `MyApp.Integrations.ExampleAPI.Mock` - Hammox mock used in tests

Production code should call the facade only.

## Why

This keeps the integration boundary explicit and stable:

- callers depend on one app-owned contract instead of a vendor client
- tests swap the implementation once in config instead of mutating global state per test
- the adapter stays generic and reusable
- business logic stays in the feature that uses the integration, not in the HTTP client

This is the standard Mox facade pattern, but use Hammox for the mock layer so callback arguments and return values are checked against the behaviour contract.

## Mocking Comparison

Use this table when choosing the testing boundary for an external integration:

| Tool | What it mocks well | Main weakness for integration boundaries | When to use it | Why this guide does not standardize on it |
| --- | --- | --- | --- | --- |
| `Mimic` | Concrete modules and low-level libraries such as `Req`, `Tesla`, or SDK clients | It does not force an explicit behaviour boundary, so tests can couple directly to transport details or vendor modules | Legacy code, migration steps, or rare transport-level implementation tests | It is too easy to skip the app-owned contract and test the wrong boundary |
| `Mox` | Behaviour-backed facades with explicit contracts | It verifies calls and arity, but it does not type-check mock inputs and outputs against the callback spec | General boundary mocking when you want the standard Elixir behaviour/proxy pattern | It is better than direct module mocking, but it can still allow fake-green tests when mocks return the wrong shape |
| `Hammox` | The same behaviour-backed facade pattern as `Mox`, plus typed expectations, stubs, and protected helpers | It requires clearer specs and slightly more discipline in callback design | Default choice for external system boundaries | It keeps the Mox architecture but adds contract checking, which is the main thing we want |

## Why Hammox Wins Here

For external systems, the highest-value failure is contract drift:

- the callback spec changes but the mock still returns the old shape
- a helper stub returns data that production could never return
- a feature test passes because the mock is "close enough" but not actually valid

`Hammox` is the best fit because it preserves the official Mox pattern while making those failures visible earlier.

## Legacy Codebases

In a legacy codebase, do not wait for a full rewrite before adopting this pattern.

Use the smallest safe migration:

1. Identify the concrete external module currently called by feature code.
2. Introduce a facade module with callbacks that describe the contract the app actually wants.
3. Move the existing implementation behind `MyApp.Integrations.ExampleAPI.Impl`.
4. Update callers to depend on the facade instead of the concrete vendor client.
5. Add a `Hammox` mock and switch caller tests to mock the facade boundary.
6. Keep `Mimic` only in any remaining implementation-detail tests until those tests can also be simplified.

The goal is not to eliminate all legacy tests in one go. The goal is to stop adding new direct dependencies on vendor modules and to make the contract clearer with each touched integration.

## Standard Shape

Define the contract and the dispatch point in the facade:

```elixir
defmodule MyApp.Integrations.ExampleAPI do
  @type request :: %{id: String.t()}
  @type response :: %{status: :ok | :error}
  @type error :: :not_configured | {:api_error, term()}

  @callback fetch_status(request()) :: {:ok, response()} | {:error, error()}

  def fetch_status(request), do: impl().fetch_status(request)

  defp impl do
    Application.get_env(:my_app, :example_api, MyApp.Integrations.ExampleAPI.Impl)
  end
end
```

Keep the implementation focused on transport and translation:

```elixir
defmodule MyApp.Integrations.ExampleAPI.Impl do
  @behaviour MyApp.Integrations.ExampleAPI

  alias MyApp.Integrations.ExampleAPI

  @impl ExampleAPI
  def fetch_status(request) do
    with {:ok, api_key} <- fetch_api_key(),
         {:ok, raw_response} <- post_request(request, api_key) do
      {:ok, normalize_response(raw_response)}
    end
  end
end
```

## Boundary Rules

- The facade owns the behaviour, shared types, and implementation lookup.
- The implementation owns HTTP calls, auth headers, retries, and response normalization.
- The feature code owns prompts, business rules, fallback policy, and user-facing error handling.
- The implementation should not know about a specific screen, workflow, or product story.
- Avoid calling `Application.get_env/3` from every caller. The facade should be the only dispatch point.

## Testing Pattern

Define the mock once, usually in `test_helper.exs`:

```elixir
Hammox.defmock(MyApp.Integrations.ExampleAPI.Mock, for: MyApp.Integrations.ExampleAPI)
```

Point the test environment at the mock once:

```elixir
config :my_app, :example_api, MyApp.Integrations.ExampleAPI.Mock
```

In caller tests, mock the facade contract, not the HTTP client:

```elixir
defmodule MyApp.SomeFeatureTest do
  use ExUnit.Case, async: true

  import Hammox

  setup :verify_on_exit!

  test "handles a successful external response" do
    expect(MyApp.Integrations.ExampleAPI.Mock, :fetch_status, fn %{id: "123"} ->
      {:ok, %{status: :ok}}
    end)

    assert :ok = MyApp.SomeFeature.run("123")
  end
end
```

Test the real adapter separately by calling `MyApp.Integrations.ExampleAPI.Impl` directly and mocking the lower transport boundary there.

## Hammox-Specific Guidance

- Prefer `import Hammox` in tests that set expectations or stubs.
- Use `setup :verify_on_exit!` anywhere expectations are set.
- Use `setup :set_mox_from_context` and `allow/3` when the mocked calls happen in spawned processes.
- Use `stub_with/2` when a shared default stub keeps tests smaller and clearer.
- Use `protect/2` or `protect/3` for hand-written stub modules or helper functions when you want them checked against the behaviour contract.

## Avoid

- calling vendor clients directly from contexts, LiveViews, controllers, or jobs
- embedding feature-specific prompts or workflow decisions in the adapter
- adding new `Mimic`-based boundary tests for code that could be behind a behaviour
- swapping implementations inside individual tests unless the test is explicitly about missing config or dispatch
- defining ad-hoc mock modules per test
- letting callers depend on raw third-party response shapes

## Review Questions

- Does the external system have one clear app-owned contract?
- Do all callers go through the facade?
- In legacy areas, did this change move the code closer to the facade pattern instead of further away from it?
- Is the implementation generic, or is it quietly accumulating product logic?
- Are tests mocking the behaviour boundary instead of the vendor client?
- Would these tests still fail if the mock returned the wrong shape or wrong types?
- Is the test environment configured once, rather than patched repeatedly in test bodies?
