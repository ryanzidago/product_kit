# Business Logic Computation

## Rule

Model non-trivial business computations as explicit, composable, mostly pure steps.

Keep persistence, input validation, and side effects at the boundary.

Use a computation struct with appended step records when the computation is large enough to benefit from composition, introspection, and explainability.

## When This Applies

This guide applies to calculation-heavy domain logic such as:

- holiday and entitlement calculations
- annualised hours calculations
- Bradford factor or absence scoring
- pricing, accrual, and balance calculations
- policy-driven eligibility or threshold rules
- report or export values derived from multiple business inputs

If the logic would still matter if the UI, job runner, or API changed, it belongs in this shape.

## Why

Calculation-heavy business logic tends to go wrong in the same ways:

- persistence gets mixed into the calculation and makes tests slow and brittle
- validation, normalization, and calculation get collapsed into one function
- the code only works one record at a time even though the domain naturally batches
- the result is hard to explain because intermediate decisions are hidden
- a bug fix in one branch quietly changes another branch because the steps are not explicit

The goal is not just to get the final number.

The goal is to make the calculation:

- correct
- testable
- batch-friendly
- explainable
- reusable across LiveViews, jobs, APIs, and exports

## Core Shape

Prefer this flow:

1. load the required records and reference data
2. validate and normalize the input shape
3. resolve the owning datetime boundaries and timezone context
4. run pure or mostly pure computation steps
5. persist or publish the result at the edge

Prefer:

```elixir
def recompute_records(record_ids, start_datetime) do
  records = Records.list_for_recalculation(record_ids)
  reference_data = ReferenceData.for_records(records, start_datetime)
  ruleset = Rulesets.fetch!(start_datetime)
  timezone = Organisations.shared_timezone!(records)

  with {:ok, computation} <-
         BusinessComputation.run(records, %{
           start_datetime: start_datetime,
           reference_data: reference_data,
           ruleset: ruleset,
           timezone: timezone
         }) do
    ComputationResults.persist(computation.current)
  end
end
```

Avoid:

```elixir
def recompute_records(record_ids, start_datetime) do
  Enum.each(record_ids, fn record_id ->
    record = Repo.get!(Record, record_id)
    reference_data = Repo.all(from item in ReferenceItem, where: item.scope == ^record.scope)
    ruleset = Repo.get_by!(Ruleset, effective_on: start_datetime)

    if record.input_a && record.input_b do
      computed_value =
        apply_business_rules(record, reference_data, ruleset)

      record
      |> Record.changeset(%{computed_value: computed_value})
      |> Repo.update!()
    end
  end)
end
```

The avoided shape mixes:

- data loading
- validation
- time and policy lookup
- the actual computation
- persistence

That makes the core logic harder to test, reuse, and explain.

## Separate Validation From Computation

Validation answers whether the input is acceptable.

Computation answers what the result should be.

Do not spread required-field checks, parsing, and fallback defaults across every computation step.

Prefer:

- parse and normalize raw inputs once
- reject impossible or incomplete input before the main computation starts
- let the core computation operate on domain-shaped values

Prefer:

```elixir
with {:ok, normalized_input} <- Input.normalize(raw_input),
     :ok <- Input.validate(normalized_input) do
  BusinessComputation.compute(normalized_input)
end
```

Avoid:

```elixir
def compute(raw_input) do
  weekly_hours = Map.get(raw_input, "weekly_hours", 0)
  weeks_per_year = Map.get(raw_input, "weeks_per_year", 52)
  timezone = Map.get(raw_input, "timezone", "UTC")

  ...
end
```

Hidden defaults inside the calculation make the output hard to trust and hard to audit later.

## Keep The Computation Core Pure

The core computation should be deterministic from its inputs.

Avoid inside the core:

- database reads or writes
- network calls
- process dictionary state
- hidden `DateTime.utc_now/0`
- hidden config lookups
- logging as control flow

Prefer passing these in explicitly:

- policy version or rule set
- reference data and lookup tables
- timezone ownership
- the relevant start datetime or explicit business boundary
- preloaded reference data

This is especially important for computation-heavy logic because "today", local business dates, reference data, and effective policy versions often change the result.

See [elixir/time-handling.md](../elixir/time-handling.md) for time and timezone rules.
See [elixir/function-purity.md](../elixir/function-purity.md) for the broader boundary between pure logic and effectful workflow code.

## Use Step-Oriented Computation Structs

For multi-stage calculations, prefer threading one computation struct through plain functions.

This is similar in spirit to [`Ecto.Multi`](https://hexdocs.pm/ecto/Ecto.Multi.html):

- you keep appending named operations to a data structure
- order is explicit
- composition is straightforward
- failures can point at a specific step
- partial progress can be inspected

But it is not a database transaction tool and should not pretend to be one.

The point is explicit computation structure, not SQL atomicity.

You usually do not need a separate pipeline-builder abstraction for this. Plain Elixir functions and one threaded struct are enough.

Prefer:

```elixir
defmodule MyApp.BusinessComputation do
  alias MyApp.Computation

  def run(records, opts) do
    records
    |> Computation.new(%{
      start_datetime: opts.start_datetime,
      reference_data: opts.reference_data,
      ruleset: opts.ruleset,
      timezone: opts.timezone
    })
    |> resolve_business_boundary()
    |> normalize_inputs()
    |> apply_rules()
    |> build_result_rows()
  end
end
```

## Append Explanation At The Same Time As The Value

Each computation function should compute one step and append one step record.

Build the explanation at the same moment as the value so the reasoning does not have to be reconstructed later.

Prefer:

```elixir
defp normalize_inputs(%Computation{} = computation) do
  normalized_inputs = normalize(computation.current, computation.context)

  step = %Computation.Step{
    name: :normalized_inputs,
    value: normalized_inputs,
    explanation: %{
      dropped_records: dropped_record_ids(normalized_inputs),
      normalization_rules: [:trim_strings, :coerce_decimals]
    }
  }

  Computation.append_step(computation, step)
end
```

The same pattern should hold for later functions such as `apply_rules/1` and `build_result_rows/1`.

## Keep The Module Layout Small

Prefer a small, explicit layout when this pattern grows beyond one file:

```text
MyApp.BusinessComputation
MyApp.BusinessComputation.Computation
MyApp.BusinessComputation.Computation.Step
```

Typical ownership:

- `MyApp.BusinessComputation` - public entry point and step functions
- `MyApp.BusinessComputation.Computation` - the threaded computation struct and helper functions such as `new/2` and `append_step/2`
- `MyApp.BusinessComputation.Computation.Step` - the step struct used for values and explanations

Do not split this into many files too early. The point is to keep the computation explicit, not to create a framework around it.

## Use A Small Canonical Computation Shape

Prefer one small, stable internal result shape so computation modules, tests, and debugging output all use the same vocabulary.

Prefer:

```elixir
%Computation{
  input: records,
  context: %{
    start_datetime: ~U[2026-04-01 00:00:00Z],
    timezone: "Europe/London",
    ruleset_version: "2026-01"
  },
  current: [%{id: "a1", computed_value: Decimal.new("1742")}],
  steps: [
    %Computation.Step{
      name: :business_boundary,
      value: %{day_starts_at: ~U[2026-03-31 23:00:00Z]},
      explanation: %{boundary_source: :organisation_timezone}
    },
    %Computation.Step{
      name: :normalized_inputs,
      value: [%{id: "a1", input_a: Decimal.new("12")}],
      explanation: %{normalization_rules: [:trim_strings, :coerce_decimals]}
    },
    %Computation.Step{
      name: :computed_values,
      value: [%{id: "a1", value: Decimal.new("1742")}],
      explanation: %{applied_rules: [:base_rule, :cap, :round_half_up]}
    },
    %Computation.Step{
      name: :result_rows,
      value: [%{id: "a1", computed_value: Decimal.new("1742")}],
      explanation: %{row_count: 1}
    }
  ],
  error: nil
}
```

On failure, prefer the same shape with explicit step ownership:

```elixir
{:error,
 %Computation{
   current: [%{id: "a1", input_a: Decimal.new("12")}],
   steps: [
     %Computation.Step{
       name: :business_boundary,
       value: %{day_starts_at: ~U[2026-03-31 23:00:00Z]},
       explanation: %{boundary_source: :organisation_timezone}
     },
     %Computation.Step{
       name: :normalized_inputs,
       value: [%{id: "a1", input_a: Decimal.new("12")}],
       explanation: %{normalization_rules: [:trim_strings, :coerce_decimals]}
     }
   ],
   error: %{
     step: :computed_values,
     reason: {:missing_reference_data, :rate_table}
   }
 }}
```

Avoid returning only the final scalar or a vague failure:

```elixir
{:ok, [%{id: "a1", computed_value: Decimal.new("1742")}]}
```

```elixir
{:error, :calculation_failed}
```

This shape should stay small.

It does not need to capture every private intermediate variable. It needs to capture the named step results and failure point that another engineer, a test, or a support workflow can actually use.

If this computation sits behind a broader application boundary, translate it there into the public result shape the caller should consume.

## Why This Shape Pays Off

This structure makes the computation easier to test:

- unit tests can call one function such as `normalize_inputs/1` or `apply_rules/1` directly
- unit tests can assert one step record without setting up the whole write path
- scenario tests can assert both the final result and the explanation data
- most correctness tests can stay out of the database because the computation takes plain input data

It makes the computation easier to debug and introspect:

- a failure identifies the exact step that broke
- partial progress is still visible in `steps`
- support and engineers can inspect why a result was produced without replaying the whole workflow by hand
- explanations are built when the value is produced, so they do not drift from the actual computation

It makes the computation easier to change:

- adding, removing, or reordering a function is explicit
- small step functions stay easy to review and refactor
- new output fields can be introduced without rewriting the storage layer first

It also keeps the database boundary cleaner:

- loading and writing can evolve separately from the rule engine
- the same computation can power previews, audits, backfills, and persisted writes
- heavy correctness tests do not need `Repo`, transactions, or factory setup unless the behavior truly depends on persistence

## Step Outputs Should Be Explainable

Complex business calculations should not return only the final number when users, support, or auditors may ask why.

Prefer step outputs that make the decision chain visible.

Examples:

- normalized inputs and exclusions
- boundary or cutoff used
- rules applied and why
- intermediate totals, balances, or scores
- caps, floors, or rounding rules applied

Prefer:

```elixir
%{
  result: %{computed_value: Decimal.new("1742")},
  explanation: %{
    boundary_used: ~U[2026-04-01 00:00:00Z],
    ruleset_version: "2026-01",
    applied_rules: [:base_rule, :cap, :round_half_up],
    intermediate_values: %{
      base_value: Decimal.new("1950"),
      deduction: Decimal.new("208")
    }
  }
}
```

Avoid:

```elixir
%{result: %{computed_value: Decimal.new("1742")}}
```

The final number alone is often not enough to debug, review, or explain a disputed result.

## Design For Bulk Operations Early

Many business computations naturally run over sets:

- all records affected by a rule change
- all entities in a reporting or processing window
- all inputs that depend on shared reference data
- all contracts, balances, or scores effective on a cutoff date

Do not force a batch domain into a one-row-at-a-time API unless the domain truly needs per-record interactive behavior.

Prefer:

- accepting a collection when the workflow is batch-shaped
- loading shared reference data once
- computing in memory across the full set
- persisting in bulk when the write shape fits

Prefer:

```elixir
def run(records, opts) do
  records
  |> Enum.map(&compute_one(&1, opts))
  |> collect_rows_and_explanations()
end
```

Avoid:

```elixir
Enum.each(records, fn record ->
  Repo.update!(Record.changeset(record, compute_one(record, opts)))
end)
```

Bulk-first design usually gives you:

- fewer queries
- fewer repeated policy and calendar lookups
- simpler scheduling for jobs and backfills
- more predictable performance

When the final persistence shape is a true bulk write, follow [elixir/ecto/bulk-operations.md](../elixir/ecto/bulk-operations.md).

## Persist At The Edge

The computation module should usually return computed rows, decisions, and explanations.

Another boundary should decide how to persist or publish them.

Examples of edge work:

- `insert_all` or `update_all`
- writing audit rows
- enqueueing follow-up jobs
- publishing events
- mapping the result into UI or export output

This keeps the computation reusable for:

- previews before save
- dry runs
- backfills
- audits
- exports
- user-facing explanations

## Time, Reference Data, And Policy Versions Must Be Explicit

Calculation-heavy business rules often depend on:

- which timezone owns the business day
- which reference data set applies
- which policy version was effective
- which start datetime owns the calculation boundary

Do not hide those decisions behind ambient defaults.

Default to explicit datetimes for these inputs unless the value is irreducibly date-only.

Prefer:

```elixir
%{
  start_datetime: ~U[2026-04-01 00:00:00Z],
  timezone: "Europe/London",
  reference_data_version: "2026-04",
  policy_version: "2026-01"
}
```

Avoid:

```elixir
opts = %{}
```

If the result depends on time or policy ownership, those inputs are part of the contract.

If a caller starts with a `Date`, resolve it into the owned datetime boundary before the main computation starts rather than letting each step guess.

## Testing

Prefer a layered test shape:

- small tests for individual computation steps
- realistic end-to-end computation scenarios with production-shaped data
- edge-case tests around boundaries such as cutoffs, caps, rounding, leap years, and reference-data changes
- explicit tests for explanation output when explainability matters

Pure computation modules should be fast enough that most correctness tests do not need the database.

The main payoff should be visible in the test suite:

- most rule coverage lives in plain-data tests
- only a smaller set of tests need `Repo` to prove loading and persistence boundaries
- explanation output is asserted alongside the value that produced it

### Pure Computation Test

Prefer plain-data tests for the core rules:

```elixir
test "run/2 computes values and records explanations without touching the database" do
  records = [
    %{id: "a1", input_a: Decimal.new("12"), input_b: Decimal.new("4")}
  ]

  {:ok, computation} =
    BusinessComputation.run(records, %{
      start_datetime: ~U[2026-04-01 00:00:00Z],
      timezone: "Europe/London",
      ruleset: %{version: "2026-01", cap: Decimal.new("2000")},
      reference_data: %{rate_table: %{default: Decimal.new("145.1667")}}
    })

  assert computation.current == [
           %{id: "a1", computed_value: Decimal.new("1742")}
         ]

  assert Enum.map(computation.steps, & &1.name) == [
           :business_boundary,
           :normalized_inputs,
           :computed_values,
           :result_rows
         ]

  computed_values_step =
    Enum.find(computation.steps, &(&1.name == :computed_values))

  assert computed_values_step.explanation == %{
           applied_rules: [:base_rule, :cap, :round_half_up]
         }
end
```

This is where most correctness should live. It is fast, explicit, and does not need fixtures, factories, or transaction setup just to prove the business rule.

### Database Boundary Test

Use a smaller number of database-backed tests to prove loading and persistence boundaries:

```elixir
test "recompute_records/2 loads inputs and persists computed rows" do
  record = insert!(:record, input_a: Decimal.new("12"), input_b: Decimal.new("4"))
  insert!(:reference_item, scope: record.scope, key: :default_rate, value: Decimal.new("145.1667"))
  insert!(:ruleset, effective_on: ~U[2026-04-01 00:00:00Z], version: "2026-01")

  assert :ok = recompute_records([record.id], ~U[2026-04-01 00:00:00Z])

  persisted_result =
    Repo.get_by!(ComputationResult, source_record_id: record.id)

  assert persisted_result.computed_value == Decimal.new("1742")
end
```

This test should prove the boundary wiring:

- the right records are loaded
- the computation is invoked with the expected context
- the computed result is persisted in the expected shape

It should not carry the full burden of proving every calculation rule. That belongs in the pure computation tests.

### Failure-Step Test

Use pure tests to prove that failures stay inspectable at the step boundary:

```elixir
test "run/2 records the failing step when required reference data is missing" do
  records = [
    %{id: "a1", input_a: Decimal.new("12"), input_b: Decimal.new("4")}
  ]

  assert {:error, computation} =
           BusinessComputation.run(records, %{
             start_datetime: ~U[2026-04-01 00:00:00Z],
             timezone: "Europe/London",
             ruleset: %{version: "2026-01", cap: Decimal.new("2000")},
             reference_data: %{}
           })

  assert computation.error == %{
           step: :computed_values,
           reason: {:missing_reference_data, :rate_table}
         }

  assert Enum.map(computation.steps, & &1.name) == [
           :business_boundary,
           :normalized_inputs
         ]
end
```

This is one of the main advantages of the pattern. The failure is not just "the computation failed". The test can prove exactly where it failed and what had already been computed.

See [testing/abstract-vs-real-world.md](../testing/abstract-vs-real-world.md) for the abstract-versus-realistic test split.

## Review Questions

- Is persistence mixed into the core calculation?
- Are validation and computation collapsed into one hard-to-test function?
- Are time, reference-data, and policy inputs explicit?
- Could this API work over a collection instead of one row at a time?
- Can the caller inspect named intermediate results or only the final number?
- If a result is disputed, can we explain how it was produced?
- Would this computation still be usable from a job, API, export, and preview flow?
