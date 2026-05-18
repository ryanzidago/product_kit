# Background Jobs

## Rule

Use background jobs for work that should happen outside the request or LiveView process, and keep the worker thin.

The worker should primarily:

- accept a small, durable argument payload
- load the current state it needs
- call a feature module that owns the real business logic
- translate the result into Oban job outcomes

This applies whether the module uses `Oban.Worker` or `Oban.Pro.Worker`.

## When To Use A Job

Use a background job when at least one of these is true:

- the work is slow enough to hurt request or LiveView latency
- the work needs retries after transient failure
- the work depends on external systems or unreliable network calls
- the work should survive node restarts
- the work is fan-out or batch orchestration
- the work is scheduled or recurring

Do not create a job just to avoid writing a clean synchronous flow. If the work is fast, local, and part of the user-facing transaction, keep it inline.

## Keep Workers Thin

Put domain behavior in app modules, not in the worker callback.

Prefer an ordinary job parameter in the function head and extract required args explicitly in the body. Use pattern matching in the function head to guard that required keys are present (clause selection), then extract values in the body. Add a fallback clause that discards jobs with missing args.

Prefer this shape:

```elixir
defmodule MyApp.Billing.Workers.SyncInvoiceWorker do
  use Oban.Worker,
    queue: :billing,
    max_attempts: 5,
    unique: [fields: [:args], keys: [:invoice_id], period: 60]

  alias MyApp.Billing.InvoiceSync

  @spec enqueue(Ecto.UUID.t()) :: {:ok, Oban.Job.t()} | {:error, Ecto.Changeset.t()}
  def enqueue(invoice_id) do
    %{invoice_id: invoice_id}
    |> new()
    |> Oban.insert()
  end

  @impl Oban.Worker
  def perform(%Oban.Job{args: %{"invoice_id" => _}} = job) do
    %{"invoice_id" => invoice_id} = job.args

    case InvoiceSync.run(invoice_id) do
      :ok ->
        :ok

      {:error, :rate_limited} ->
        {:error, :rate_limited}

      {:error, :invoice_missing} ->
        {:cancel, :invoice_missing}
    end
  end

  def perform(%Oban.Job{}), do: {:discard, :missing_required_args}
end
```

Avoid:

- querying half the system and implementing the whole workflow in the worker
- passing sockets, conns, preloaded structs, or large rendered payloads in job args
- coupling the worker directly to a specific UI concern

The architecture guide still applies here: `presentation -> business logic -> database operations`.

## Job Args

Pass the smallest durable payload that lets the job recover and retry safely:

- record IDs
- tenant or organisation IDs when required for scoping
- trigger metadata that matters for auditing or branching
- idempotency or deduplication keys when needed

Prefer reloading current state inside the job over serializing full records into args.

This keeps retries correct when the underlying data changes between enqueue time and execution time.

Treat job args as a compatibility boundary:

- prefer adding new optional keys over changing the meaning of existing keys
- avoid renaming or removing keys while older jobs may still be queued
- assume deploys can leave old jobs in the queue running against newer code

If you need a breaking change, add a new worker or support both argument shapes during the transition.

## Enqueueing

Expose explicit enqueue helpers on the worker module, such as `enqueue/1`, `enqueue/2`, or a clearly named orchestration helper.

When a job depends on a database write that must commit first, enqueue it in the same transaction:

```elixir
Ecto.Multi.new()
|> Ecto.Multi.update(:account, changeset)
|> Oban.insert(:sync_job, fn %{account: account} ->
  SyncAccountWorker.new(%{account_id: account.id})
end)
|> Repo.transact()
```

Use this pattern when the job should exist if and only if the write commits.

For fan-out work:

- use an orchestrator worker to find the work
- enqueue per-record jobs with `Oban.insert_all/1` when batching is clearer or cheaper
- keep each child job focused on one durable unit of work

For scheduled work:

- keep the scheduled worker as an orchestrator where possible
- fan out into ordinary jobs for tenant-, organisation-, or record-level processing

## Timeouts And Cancellation

Set worker timeouts deliberately.

Use the default timeout when it matches the workload. Override it when the job legitimately needs longer, but treat that as a signal to check whether the job should be split into smaller units first.

Do not let jobs occupy queue slots indefinitely:

- fail fast when the target record is gone or the work is no longer relevant
- return `{:cancel, reason}` or `{:discard, reason}` for hopeless work
- reserve retries for failures that may succeed later

Long-running jobs should usually report progress through domain state instead of appearing stuck until the final step.

## Outcome Semantics

Choose the return value based on what retrying would accomplish:

- malformed or incomplete job args: usually `{:discard, reason}`
- valid job, but the target record or state is no longer relevant: usually `{:cancel, reason}`
- transient failure that may succeed later: `{:error, reason}`
- completed work: `:ok`

Retries rarely fix a job that was enqueued with the wrong payload, so malformed args should usually not be retried.

## Idempotency And Deduplication

Assume every job can run more than once.

Retries, crashes, timeouts, and manual re-enqueueing all make duplicate execution possible. Design the side effects so a second run is harmless or detected cleanly.

Use Oban `unique` options when duplicate enqueueing is likely and the business case wants deduplication. Good unique keys usually reflect the true business identity of the work, such as:

- invoice ID
- organisation ID plus trigger source
- tenant ID plus external resource ID

Do not rely on uniqueness alone for correctness. Uniqueness reduces duplicate inserts; idempotent business logic protects the side effects.

For external side effects, prefer an app-owned idempotency key when the remote system supports one. A retry-safe local job is not enough if each retry creates a second remote charge, email, or webhook effect.

## Queues And Concurrency

Use a dedicated queue when the work has distinct concurrency, priority, or operational characteristics.

Examples:

- chat or user-facing async work
- external integrations
- maintenance or integrity checks
- long-running batch jobs

When ordering matters for one business entity, partition concurrency by the business key if the queue engine supports it. This is the right tool when the same record must be processed sequentially but different records can proceed in parallel.

Watch for backpressure:

- queue depth growing faster than workers can drain it
- retries consuming capacity meant for fresh work
- one noisy workload starving unrelated jobs

When this happens, split queues by workload, lower per-job scope, or adjust concurrency based on the real bottleneck rather than adding retries blindly.

## Operational Defaults

Prefer these defaults unless the workload has a clear reason to differ:

- assume at-least-once execution semantics
- make duplicate execution safe in domain logic, not only at enqueue time
- treat already-applied work as success when possible
- retry only transient failures such as timeouts, rate limits, temporary dependency outages, or safe-to-retry concurrency races
- discard or cancel permanent failures such as malformed args, forbidden work, unsupported transitions, or stale targets that no longer need processing
- use Oban uniqueness to reduce duplicate inserts, not as the only correctness guarantee
- start with concurrency that matches the real bottleneck such as database capacity, vendor rate limits, or tenant hot spots
- split queues or lower per-job scope before raising concurrency blindly
- serialize by business key only when ordering matters for that entity; do not serialize the whole workload by default

For flaky external integrations, prefer explicit, slower backoff over a burst of immediate retries:

```elixir
defmodule MyApp.Billing.Workers.SyncInvoiceWorker do
  use Oban.Worker, queue: :billing, max_attempts: 5

  @impl Oban.Worker
  def backoff(%Oban.Job{} = job), do: job.attempt * 60
end
```

For idempotency, prefer domain behavior that turns duplicate execution into a safe no-op:

```elixir
case InvoiceSync.run(invoice_id) do
  :ok ->
    :ok

  {:ok, :already_synced} ->
    :ok

  {:error, :rate_limited} ->
    {:error, :rate_limited}

  {:error, :invoice_missing} ->
    {:cancel, :invoice_missing}
end
```

If the same business entity must be processed sequentially, constrain concurrency at that entity boundary rather than globally. If ordering does not matter for that entity, prefer independent jobs and idempotent side effects over broad serialization.

## Multi-Tenant Safety

When work is tenant-, organisation-, or account-scoped, include that scope explicitly in the args when the worker needs it.

Do not assume an ID alone is always safe across tenants or sufficient for policy checks. Reload the target data within the right scope and re-check the business rules that still matter at execution time.

This matters especially for jobs that:

- call external systems on behalf of a tenant
- run after permissions or membership may have changed
- operate on records whose visibility is scoped by organisation or tenant

## Result Semantics

Return the Oban outcome that matches the failure mode:

- `:ok` when the work completed
- `{:error, reason}` for retryable failures
- `{:discard, reason}` for non-retryable failures that should stop retrying
- `{:cancel, reason}` when the job should not run because the target state no longer exists or no longer needs processing

Be deliberate here. Treating permanent failures as retryable creates noisy dead jobs and wasted work.

## Data Freshness And Intent

Be explicit about whether the job should use current state or preserve enqueue-time intent.

Prefer current-state reloads when the job is acting on a record that may have changed and the latest truth should win.

Preserve enqueue-time inputs only when the business action itself is the thing to remember, for example:

- the exact export window requested by a user
- the payload accepted from an upstream webhook
- a snapshot identifier produced earlier in a workflow

If preserving intent matters, store that intent explicitly in the args or on a domain record. Do not rely on re-deriving it later from changed data.

## Side Effects And State

If the UI or downstream systems need progress visibility, persist or broadcast domain state explicitly. Do not make callers infer everything from raw Oban job state.

Good patterns include:

- updating a domain record status before and after key steps
- storing audit metadata
- broadcasting PubSub events for user-visible progress

Oban tracks job execution. It should not be your only source of product state.

If the product shows job progress to users, model that state in app terms such as `queued`, `processing`, `completed`, or `failed` on a domain record the UI already understands.

## Observability

Make background work observable at both the job level and the business level.

At minimum:

- log the worker name and business identifiers such as tenant ID, organisation ID, or record ID
- emit or monitor metrics for enqueue volume, execution latency, retries, failures, and dead jobs
- make queue health visible enough that backlog and starvation are noticed early

Logs should help answer both:

1. Which job failed?
2. Which business entity or workflow did that failure affect?

## Dead Jobs And Manual Recovery

Assume some jobs will end up dead even in a healthy system.

Document or encode the recovery policy:

- who owns dead jobs for this queue
- which failures are safe to retry manually
- which failures should stay dead because the underlying state is stale or invalid

Retrying dead jobs without understanding the failure mode often recreates the same incident.

## Cron And Scheduled Work

Keep cron-triggered workers thin.

Prefer a scheduler or orchestrator job that discovers the relevant work and enqueues ordinary jobs for the actual processing. This keeps schedule concerns separate from business execution and makes ad hoc re-runs easier.

Scheduled jobs should have clear ownership, expected frequency, and a known effect when a run is delayed or skipped.

## Deploy Safety

Assume code and queued jobs can span a deploy boundary.

That means:

- queued jobs may have been built with the previous code version
- workers may execute while data migrations are still rolling out
- retries may happen long after the original enqueue

Avoid deploys that require all queued jobs to disappear instantly. Prefer additive changes, compatibility windows, and worker code that can handle both old and new argument shapes while the queue drains.

## Testing

Test background jobs at the right level:

- use `Oban.Testing` in tests that assert jobs were enqueued
- assert on `worker`, `queue`, and important args
- test the worker callback directly for success, retryable errors, and permanent failures
- drain queues only when you genuinely need to exercise queued execution or recursive workflows
- keep core business behavior tested outside the worker too

When the worker executes mocked integrations in another process, set up the mock ownership correctly for async execution. The test should prove the real process boundary, not only a same-process happy path.

Prefer both:

1. small enqueue tests that prove the right job was created
2. realistic execution tests that prove the worker behaves correctly with production-shaped data

## Avoid

- putting long chains of business logic directly in the worker
- hiding required side effects inside `after_process` hooks when they are core business behavior
- passing stale snapshots of records in args when IDs would do
- enqueuing jobs outside the transaction when the write and enqueue must be atomic
- using one generic queue for unrelated workloads by default
- assuming retries are rare enough to ignore idempotency

## Review Questions

- Is this work actually a good fit for a background job?
- Does the worker stay thin, with business logic living elsewhere?
- Are the args minimal, durable, and safe for retries?
- If the job depends on a write, is enqueueing atomic with that write?
- Is duplicate execution safe?
- Does the queue choice reflect the workload's concurrency and priority?
- Are tests proving both enqueue behavior and real execution behavior?
