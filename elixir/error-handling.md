# Error Handling

## Rule

Expected failures should be explicit, translated at the right boundary, and surfaced in a shape the caller can act on.

Default to `{:ok, value}` and `{:error, reason}` at application boundaries.

Reserve raises for programmer errors, impossible states, and deliberate bang APIs.

## Why

Unclear error handling creates three expensive failure modes:

- callers cannot tell what to show, retry, or log
- low-level details leak across module boundaries and couple unrelated code
- real defects get hidden behind generic fallbacks or silent success

The goal is not "never fail". The goal is to fail in a way that preserves intent, diagnosability, and correct product behavior.

## Separate Expected Failures From Defects

Treat these as expected failures:

- invalid user input
- authorization denial
- missing or stale records
- uniqueness or state conflicts
- timeouts, rate limits, and temporary dependency outages
- business rules that legitimately reject the action

Treat these as defects:

- impossible internal states
- violated assumptions between internal modules
- missing required internal data that should already exist
- code paths that should be unreachable

Expected failures should usually be returned as data.

Defects should usually fail fast so they are visible and fixable.

Do not rescue a defect and pretend the operation succeeded.

## Default Boundary Contract

For public context functions, feature orchestration modules, and integration facades, default to:

```elixir
@spec run(input()) :: {:ok, result()} | {:error, reason()}
```

Keep the error reason:

- app-owned
- stable enough for callers to branch on
- specific enough to drive the next action

Prefer app-owned structured reasons over freeform strings or leaked transport details.

Prefer:

```elixir
{:error, {:not_found, :report}}
{:error, {:forbidden, :export_report}}
{:error, {:validation_failed, changeset}}
{:error, {:conflict, :schedule_overlap}}
{:error, {:dependency_unavailable, provider: :openai, reason: :timeout}}
```

Avoid:

```elixir
{:error, "something went wrong"}
{:error, %Req.TransportError{}}
{:error, %{message: "timeout", status: 504, body: body}}
```

Raw structs, maps, and library errors can be fine inside an adapter or implementation module. Avoid returning them from public context, feature, or integration-facade boundaries.

The caller should not need to understand a vendor library or transport shape just to handle failure correctly.

## Translate Errors At The Owning Boundary

Each layer should translate failures into the vocabulary it owns.

### Integration adapters

- own HTTP, SDK, auth, timeout, and transport details
- normalize those details into app-owned integration errors
- do not leak raw client exceptions or response structs to feature code

### Contexts and feature modules

- own domain meaning
- translate storage or integration failures into business-relevant reasons
- return reasons the caller can act on without knowing the implementation

### LiveViews and controllers

- own user-visible feedback
- map domain errors into field errors, inline errors, flashes, redirects, or page states
- do not invent business rules that belong in the context

### Background workers

- own retry, cancel, discard, and dead-job semantics
- translate domain errors into the right Oban outcome
- do not retry hopeless work because the domain error was left ambiguous

This follows the same boundary rule as the [architecture guide](../architecture.md): each layer should own the failure language that matches its responsibility.

## Give The Caller Enough To Act

An error contract is only useful if it tells the caller what to do next.

A caller should usually be able to answer at least one of these questions:

- should the user correct input?
- should the caller retry later?
- should the operation stop permanently?
- should the UI show a not-found, forbidden, or conflict state?
- should support or on-call investigate?

If the caller cannot tell whether a failure is validation, authorization, conflict, or temporary dependency failure, the error contract is too vague.

## Retryability Must Be Discernible

Do not collapse retryable and permanent failures into one generic reason.

Examples of usually retryable failures:

- timeout
- rate limit
- temporary dependency outage
- optimistic lock or concurrent state race when the workflow can safely retry

Examples of usually non-retryable failures:

- validation failed
- forbidden
- not found
- unsupported state transition
- malformed job args

The exact representation can vary, but the distinction must be visible at the boundary that decides retries.

For Oban workers, that usually means turning domain results into:

- `:ok` for completed work
- `{:error, reason}` for retryable failures
- `{:cancel, reason}` when the target state no longer wants the work
- `{:discard, reason}` for bad payloads or other hopeless failures

See [background-jobs.md](background-jobs.md) for worker-specific behavior.

## Use Changesets For Field-Level Input Errors

When the failure belongs to user-editable fields or database-backed constraint feedback, keep it in the changeset path where the caller is form-oriented.

Examples:

- required field missing
- invalid format
- unique constraint violation
- foreign key constraint violation

When the failure is a broader workflow or domain decision, prefer a domain error tuple instead of forcing it into a fake field error.

Examples:

- user cannot approve their own request
- export is unavailable while a payroll run is locked
- document analysis provider is temporarily unavailable

See [changesets.md](./ecto/changesets.md) for the field-validation boundary.

## Logging

Log failures with enough context to diagnose them without reproducing the issue manually.

At minimum, logs around meaningful failures should identify:

- the operation
- the relevant business identifiers such as tenant, organisation, user, or record ID
- the dependency name when an external system is involved
- whether the failure is expected or unexpected when that affects severity

Do not log:

- secrets
- tokens
- raw authorization headers
- full request bodies by default
- PII unless there is a clear, approved reason

Expected product failures such as validation or authorization denial should usually not be logged at `error` level by default. Save noisy high-severity logs for failures that need operational attention.

## LiveView And Controller Feedback

Surface errors where the user can actually recover.

Prefer:

- changeset errors for field correction
- inline action errors near the affected workflow
- explicit page error states when the page cannot continue
- retry affordances only when retry is meaningful

Avoid:

- generic "Something went wrong" messages for expected failures
- silently keeping stale UI without telling the user the action failed
- showing transport-specific or internal implementation details to the user

For unexpected failures, show a generic user-safe message and log the real context separately.

See [page-states.md](./phoenix/liveviews/page-states.md) for page-level state guidance.

## Exceptions And Rescue

Do not add broad `rescue` blocks inside ordinary feature code just to keep control flow moving.

Avoid:

- swallowing exceptions and returning `:ok`
- rescuing everything and returning `{:error, :unknown}`
- converting programmer errors into fake business failures deep inside the stack

It is acceptable to rescue at a deliberate system boundary when the boundary must protect the caller experience or normalize a third-party callback shape. When you do that:

- log enough context
- return a generic internal failure reason
- do not pretend the operation succeeded
- keep the rescue scope as narrow as possible

Bang APIs should be rare and should only exist when the caller explicitly wants exception semantics.

## Validate At System Boundaries, Trust Internal Code

Code defensively only where untrusted data enters the system:

- **API controllers** — validate params, cast types, reject malformed input
- **Webhooks** — verify signatures, validate payload shape
- **CLI** — validate args and flags before passing into core logic
- **UI / LiveView** — validate form input, sanitize user-provided data
- **External service responses** — handle unexpected shapes from third-party APIs

Once data has crossed a validated boundary, trust it. Do not add internal nil checks, type guards, or fallback clauses for cases your own code cannot produce. Those checks obscure realistic error branches and hide bugs that should crash loudly.

```elixir
# Internal domain logic — trust validated input, pattern match directly
def calculate_total(%Invoice{line_items: line_items}) do
  Enum.reduce(line_items, Decimal.new(0), fn item, acc ->
    Decimal.add(acc, item.amount)
  end)
end
```

If `line_items` is unexpectedly `nil` here, the crash points directly at the bug — which is better than a silent `|| []` fallback that hides it.

## Prefer

Translate low-level failure into domain meaning:

```elixir
def run(report_id) do
  with {:ok, report} <- Reports.fetch(report_id),
       :ok <- authorize(report),
       {:ok, export} <- Exports.generate(report) do
    {:ok, export}
  else
    {:error, :not_found} ->
      {:error, :report_not_found}

    {:error, :forbidden} ->
      {:error, :forbidden}

    {:error, {:dependency_unavailable, :s3}} ->
      {:error, :export_temporarily_unavailable}
  end
end
```

Map domain failures into UI behavior:

```elixir
socket =
  case Reports.run_export(report.id) do
    {:ok, export} ->
      push_navigate(socket, to: ~p"/exports/#{export.id}")

    {:error, {:validation_failed, changeset}} ->
      assign(socket, :form, to_form(changeset))

    {:error, :export_temporarily_unavailable} ->
      put_flash(socket, :error, "Export is temporarily unavailable. Please retry shortly.")

    {:error, :forbidden} ->
      put_flash(socket, :error, "You do not have access to export this report.")
  end

{:noreply, socket}
```

Choose the right worker outcome:

```elixir
case SyncEmployee.run(employee_id) do
  :ok ->
    :ok

  {:error, :employee_missing} ->
    {:cancel, :employee_missing}

  {:error, {:dependency_unavailable, :remote_hris}} ->
    {:error, :remote_hris_unavailable}
end
```

## Avoid

- returning raw library errors from app-facing boundaries
- using strings as the primary branching mechanism for failure handling
- mixing field validation, domain rejection, and infrastructure failure into one generic reason
- logging expected validation failures as operational incidents
- silently ignoring `{:error, reason}` values in `with` chains or async flows
- rescuing broadly inside contexts, jobs, or LiveViews to hide real defects

## Review Questions

- Does this boundary return app-owned error terms rather than transport or library details?
- Can the caller distinguish validation, authorization, conflict, not-found, and temporary dependency failure?
- If retries matter here, is retryability visible in the returned result?
- Is this check guarding against untrusted external input, or against a case that our own code should never produce?
- Are field-level input failures kept in changesets rather than disguised as generic tuples?
- Does the UI show actionable feedback for expected failures and safe generic feedback for unexpected ones?
- Are failures logged with enough business context to diagnose them without leaking secrets or PII?
- Is any `rescue` block narrowly scoped and justified by a real boundary need?
- Would a future caller know what to do next from this error contract alone?
