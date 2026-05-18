# Function Purity

## Rule

Keep side effects in small, explicit boundary functions.

Push normalization, decision-making, derivation, and result-shaping into pure functions that return the same output for the same explicit inputs.

Prefer a small impure shell around a pure core.

## Why

This gives the codebase a cleaner split:

- pure functions are cheap to test and easy to reason about
- effectful functions are easier to review because the I/O is obvious
- retries and idempotency are easier to design when the mutation points are explicit
- business rules become reusable across LiveViews, jobs, scripts, and APIs
- agents and humans can safely refactor or extend pure logic with less setup

The goal is not purity as an ideology.

The goal is to keep the parts that decide what should happen separate from the parts that make it happen in the outside world.

## What Counts As Pure Here

A pure function:

- depends only on its arguments
- does not read hidden mutable state
- does not mutate external state
- does not emit logs, telemetry, or messages as part of its normal behavior

For this repository, these are effectful and belong at the boundary by default:

- `Repo` reads and writes
- HTTP calls and vendor SDK calls
- file system reads and writes
- `DateTime.utc_now/0`, `NaiveDateTime.utc_now/0`, `System.os_time/0`, and similar clock reads
- randomness
- `Application.get_env/3`, `System.get_env/1`, and other config or environment lookups
- process dictionary access, ETS access, and global registry lookups
- PubSub broadcasts, emails, job enqueueing, and other messages to external processes or systems
- logs and telemetry emitted from deep helpers

Database reads are worth calling out explicitly.

They may not mutate state, but they still make a function depend on mutable external state. Treat them as effectful unless the already-loaded data was passed in as an argument.

## Preferred Shape

Prefer this flow:

1. gather external inputs at the boundary
2. call pure functions to normalize, validate, and decide
3. perform writes or external calls at the boundary
4. translate the outcome for the caller

Prefer:

```elixir
def update_subscription(subscription_id, params) do
  subscription = Repo.get!(Subscription, subscription_id)
  now = DateTime.utc_now()

  with {:ok, plan} <- SubscriptionUpdate.plan(subscription, params, now),
       {:ok, subscription} <- apply_subscription_update(subscription, plan),
       :ok <- Audit.log_subscription_update(subscription, plan) do
    {:ok, subscription}
  end
end
```

```elixir
defmodule MyApp.Billing.SubscriptionUpdate do
  @spec plan(Subscription.t(), map(), DateTime.t()) ::
          {:ok, map()} | {:error, term()}
  def plan(subscription, params, now) do
    normalized_params = normalize_params(params)
    changes = derive_changes(subscription, normalized_params, now)

    {:ok,
     %{
       changes: changes,
       audit_metadata: build_audit_metadata(subscription, changes, now)
     }}
  end
end
```

Avoid:

```elixir
def update_subscription(subscription_id, params) do
  subscription = Repo.get!(Subscription, subscription_id)
  now = DateTime.utc_now()
  tier = Application.fetch_env!(:my_app, :billing_tier)

  normalized_params =
    params
    |> Map.update("email", "", &String.trim/1)
    |> Map.put("tier", tier)

  changes =
    if subscription.status == :trial and now > subscription.trial_ends_at do
      %{status: :active, email: normalized_params["email"]}
    else
      %{email: normalized_params["email"]}
    end

  Logger.info("updating subscription")

  subscription
  |> Subscription.changeset(changes)
  |> Repo.update()
end
```

The avoided shape mixes state loading, clock reads, config reads, business rules, logging, and persistence into one function. That makes the logic harder to test and the side effects harder to inspect.

## Pass Inputs Explicitly

If the result depends on something, pass it in.

Common examples:

- pass `now` instead of calling `DateTime.utc_now/0` deep in the helper
- pass feature flags or selected policy versions instead of reading config in the helper
- pass already-fetched records and lookup tables instead of querying inside the helper
- pass the caller-selected locale, timezone, or tenant context instead of rediscovering it later

This keeps the function contract honest.

It also lets the caller own when external state is read and how fresh it needs to be.

See [time-handling.md](time-handling.md) for time-specific rules.

## Separate Planning From Applying

When a workflow both decides and mutates, split it into two steps:

- a pure planner that decides what should happen
- an effectful applier that performs the write, call, enqueue, or publish step

The planner can return:

- normalized attrs
- derived changes
- a command struct
- a payload for an external API
- audit or telemetry metadata

This is often clearer than having a large function that interleaves decision branches with `Repo`, HTTP, or message-sending calls.

Prefer names that make the split obvious:

- pure: `normalize_*`, `validate_*`, `derive_*`, `build_*`, `plan_*`, `present_*`
- effectful: `create_*`, `update_*`, `delete_*`, `enqueue_*`, `deliver_*`, `publish_*`, `sync_*`

Do not treat the naming as magic, but use it to help reviewers see which functions should and should not have side effects.

## Keep Observability At The Boundary

Pure helpers should not log or emit telemetry as part of their normal execution.

If a branch is important enough to observe, return enough structured information for the boundary function to log or emit telemetry there.

That keeps the helper deterministic and avoids scattering instrumentation through low-level functions.

See [observability.md](../delivery/observability.md) for the broader logging and telemetry guidance.

## Testing Consequences

This split should change the shape of the tests:

- pure functions get small deterministic tests that prove edge cases and branch behavior
- boundary functions get narrower integration tests that prove queries, writes, enqueueing, and external calls happen correctly
- broad workflow tests prove orchestration only where the workflow itself is the real risk

Do not pay database or framework cost to test logic that could have been expressed as a pure function.

Do not over-mock a boundary when the real integration seam is cheap and important to exercise.

See [../testing/test-suite-shape.md](../testing/test-suite-shape.md) for the broader testing strategy.

## Do Not Over-Abstract

Not every function needs its own purity ceremony.

Keep the split proportionate:

- a tiny formatting helper can stay inline
- a small boundary function may only need one pure helper
- a complex workflow may deserve a planner module and several pure steps

The rule is about keeping effects obvious and logic movable.

It is not a license to introduce a command pattern or extra modules for every three lines of code.

## Avoid

- reading from `Repo`, config, or the clock inside deep decision helpers
- mixing business rules and persistence in the same large function body
- logging from pure helper functions
- making changeset builders, query builders, or serializer helpers quietly hit external systems
- hiding side effects in pipelines that mostly look like data transformation
- forcing every test through a LiveView, job, or database setup when the real behavior is pure logic

## Review Questions

- What are the actual mutation points in this workflow?
- Which functions decide what should happen, and are they pure from explicit inputs?
- Is any helper quietly reading time, config, database state, or process state?
- Could this logic move into a pure function and be tested without framework setup?
- Are logs, telemetry, and side effects emitted at the boundary that owns the workflow?
