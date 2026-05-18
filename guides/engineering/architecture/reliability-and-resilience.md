# Reliability And Resilience

## Rule

Build important workflows so retries are safe, partial failures are visible, and interrupted work can be resumed, retried, or compensated without silent data loss.

Assume important work can be:

- retried
- delivered more than once
- interrupted halfway through
- delayed
- run against state that has changed since the work was queued or requested

Design for those conditions up front instead of treating them as rare edge cases.

## Why

Many production failures are not total crashes.

They are problems like:

- a job runs twice and sends two external side effects
- one step succeeds, the next fails, and the system forgets the partial success
- a timeout leaves the caller unsure whether the remote system applied the change
- work is dropped after an exception, restart, or deploy with no durable record
- retries keep hammering a permanent failure that will never succeed

These failures are expensive because they create uncertainty:

- did the work happen?
- did it happen once or multiple times?
- is it safe to retry?
- do we need to resume, reconcile, or compensate?

Reliability starts with making those answers explicit.

## When This Applies

This guide matters most for workflows such as:

- background jobs
- external API calls
- webhooks and event consumers
- imports and exports
- multi-step write workflows
- fan-out or batch processing
- user actions that trigger async side effects

Tiny local computations usually do not need this level of machinery.

Anything that crosses a process, queue, network, or time boundary usually does.

## Design For At-Least-Once Execution

Assume important work may run more than once.

That can happen because of:

- queue retries
- process crashes
- client retries
- webhook redelivery
- timeouts where the caller cannot tell whether the callee already applied the work
- manual replay or operational recovery

Do not build important workflows around an implicit exactly-once assumption unless the system can actually prove it end to end.

In most product systems, the safer default is:

- reduce duplicate delivery where practical
- make duplicate execution safe in domain logic

## Idempotency

Idempotency means a repeated attempt does not create an incorrect second effect.

This is often required for:

- charging or billing actions
- sending external updates
- webhook processing
- import row processing
- enqueue-once domain operations

Common ways to achieve it include:

- using an app-owned operation ID or idempotency key
- enforcing a uniqueness constraint on the true business identity of the action
- treating already-applied work as success instead of failure
- persisting a workflow step outcome before allowing the next step to proceed

Do not rely only on "we probably will not get duplicates."

Retries and redelivery are normal operating conditions.

## Make Partial Failure Explicit

Many important workflows are multi-step:

1. write local state
2. enqueue follow-up work
3. call an external system
4. persist the final outcome

These steps do not fail as one unit unless you have a true atomic boundary.

When atomicity is impossible:

- make the workflow steps explicit
- persist enough state to know which step completed
- make it possible to resume, retry, or compensate safely

If step two succeeds and step three fails, the system should know that.

Do not collapse partial completion into a vague "something went wrong."

## Transactional Workflows With External Side Effects

When a database write and an external side effect cannot be made atomic together, choose the simpler inconsistency on purpose.

For example, if phantom database records are worse than orphaned external objects, it can be reasonable to delete the database row first and then attempt the external delete inside the same transaction function, letting an explicit external failure return `{:error, reason}` and roll back the database delete.

```elixir
@type deleted_object :: %{
        bucket: String.t(),
        object_name: String.t()
      }

@type delete_file_error ::
        Ecto.Changeset.t()
        | :not_found
        | :forbidden
        | :timeout
        | {:gcs_error, :rate_limited | :unavailable}

@doc """
Deletes the file record first, then deletes the external object.

Failure modes:

- if `Repo.delete/1` returns `{:error, changeset}`, the external delete is not attempted
- if `GoogleStorageBucketClient.delete_object/1` returns `{:error, reason}`, the transaction returns `{:error, reason}` and rolls back the database delete
- if both steps succeed, the transaction returns both deleted values

This biases toward avoiding phantom database records without pretending the database
and storage service are jointly atomic.
"""
@spec delete_file(File.t()) ::
        {:ok, %{deleted_file: File.t(), deleted_object: deleted_object()}}
        | {:error, delete_file_error()}
def delete_file(file) do
  Repo.transact(fn ->
    with {:ok, deleted_file} <- Repo.delete(file),
         {:ok, deleted_object} <- GoogleStorageBucketClient.delete_object(file) do
      {:ok, %{deleted_file: deleted_file, deleted_object: deleted_object}}
    else
      {:error, error} ->
        {:error, error}
    end
  end)
end
```

This does not make the database and external system jointly atomic, but it is a valid pragmatic choice when the code stays simple, the tradeoff is understood, and the chosen mismatch is the cheaper one for the product.

## Choose A Batch Or Fan-Out Failure Strategy Deliberately

When processing many items, choose the failure mode on purpose.

The common options are:

- atomic all-or-nothing work
- fail-fast processing
- accumulate partial success and partial failure

### Atomic All-Or-Nothing

Use a real transaction when the work is local and must either all commit or all roll back.

This is the right fit when:

- one invalid item makes the whole batch invalid
- downstream state must never observe partial completion
- the work stays inside one true atomic boundary

Do not describe a workflow as atomic if it includes external side effects that cannot roll back with the database transaction.

### Fail-Fast

Stop on the first failure when continuing would be unsafe, misleading, or wasteful.

This is often the right fit when:

- later work depends on earlier success
- one failure means the remaining work should not proceed
- continuing would create more cleanup or more ambiguity

In Elixir, `Enum.reduce_while/3` is often a clear fit for this shape.

Fail-fast is not the same as atomic.

If earlier work already committed or produced an external side effect, aborting on the next failure does not roll it back.

### Accumulate Partial Results

Continue processing independent items when partial completion is better than total abort.

This is often the right fit when:

- the items are independent
- partial success has real product value
- the failed subset can be retried, reconciled, or reviewed later

In Elixir, `Enum.reduce/3` is often a clear fit for accumulating success and failure separately.

For smaller in-memory workflows, a shape like this can be enough:

```elixir
%{oks: [], errors: []}
```

Start with in-memory accumulation, and add persisted progress only when replaying the whole batch is too expensive, too slow, or too risky because of side effects.

When the batch does justify more operational machinery, a more durable approach can include:

- persist progress instead of holding the full result in memory
- store failed item identifiers and reasons explicitly
- retry only the failed subset
- track summary counts when the full success list is not operationally useful

The right result shape is domain-specific.

The important rule is that the workflow should make partial completion and retry boundaries explicit.

## Bound External Work With Timeouts

Every external dependency should have deliberate time bounds.

Without them:

- queue workers can occupy capacity indefinitely
- requests can appear stuck instead of failed
- one degraded dependency can spread latency through the system

Timeouts should be paired with a clear retry decision:

- retry transient failures that may succeed later
- do not retry permanent failures such as validation rejection, forbidden access, or malformed payloads

If a timeout leaves the result ambiguous, design the operation so a retry or reconciliation step is safe.

## Retry Deliberately

Retries are a resilience tool, not a universal fix.

Good retry candidates include:

- timeouts
- temporary dependency outages
- rate limits
- transient lock or concurrency races
- short-lived network failures

Bad retry candidates include:

- invalid input
- unsupported state transitions
- authorization denial
- permanently missing required data
- bugs that will fail the same way every time

Prefer bounded retries with backoff.

Immediate repeated retries often turn one dependency problem into a larger outage.

## Avoid Silent Data Loss

Important work should not disappear quietly.

If work fails, is discarded, or is skipped, there should be a durable and diagnosable signal such as:

- persisted workflow state
- a queue record
- an error event
- a structured log with stable identifiers
- a dead-letter or discard path

The exact mechanism can vary.

The important rule is that an operator or engineer should be able to answer:

- what failed?
- for which record or workflow?
- was it retried?
- is manual action needed?

Do not rescue, log a vague message, and return success for important work.

## Persist Workflow State When The Work Outlives One Request

If a workflow may continue after the request, process, or page interaction ends, persist enough state to understand and recover it later.

That often means explicit states such as:

- queued
- running
- succeeded
- failed
- cancelled
- needs_retry

Use state names that describe real operational meaning.

Avoid vague states that hide what to do next.

For long-running or multi-step work, persisted state is often more useful than trying to reconstruct history from logs alone.

## Prefer Resume Or Compensate Paths

When a workflow cannot be made atomic, choose one of these strategies explicitly:

- resume from the last safe step
- retry the failed step safely
- reconcile by checking current external and internal state
- compensate for the already-completed step

Examples:

- if an export upload fails after the file is generated, retry the upload instead of regenerating everything blindly
- if a remote side effect may already have happened, reconcile before issuing it again
- if one step cannot be undone, record that fact and prevent a misleading rollback story

The right strategy depends on the workflow.

The important part is choosing one on purpose.

## Prefer

- operation IDs, idempotency keys, or uniqueness boundaries that reflect the true business action
- explicit classification of transient versus permanent failure
- persisted workflow state for multi-step or async work
- bounded retries with backoff
- treating already-applied work as success when that matches the business meaning
- enough observability to diagnose one concrete failed execution
- domain logic that remains safe when the same work is attempted again

## Avoid

- assuming queues, clients, or webhooks deliver exactly once
- doing irreversible side effects before recording the operation identity
- retrying every failure the same way
- swallowing exceptions or converting partial failure into fake success
- depending on in-memory process state as the only record of important work
- vague workflow states that make recovery ambiguous
- using deduplication at enqueue time as the only correctness guarantee

## Example: Retry-Safe Background Job

Prefer a worker and workflow boundary where duplicate execution is expected and safe:

```elixir
defmodule MyApp.Billing.SyncInvoice do
  alias MyApp.Billing.Invoices
  alias MyApp.Integrations.InvoiceAPI

  @spec run(Ecto.UUID.t(), String.t()) :: :ok | {:error, term()}
  def run(invoice_id, operation_id) do
    with {:ok, invoice} <- Invoices.fetch_for_sync(invoice_id),
         :ok <- Invoices.ensure_not_already_synced(invoice, operation_id),
         {:ok, remote_result} <- InvoiceAPI.sync(invoice, idempotency_key: operation_id),
         {:ok, _invoice} <- Invoices.mark_synced(invoice, operation_id, remote_result) do
      :ok
    end
  end
end
```

This shape is safer because:

- the workflow has a stable operation identity
- the remote call can be retried safely
- the local state records whether this operation already succeeded

## Example: Distinguish Retryable From Permanent Failure

Prefer return values that make the next action obvious:

```elixir
case InvoiceSync.run(invoice_id, operation_id) do
  :ok ->
    :ok

  {:error, :rate_limited} ->
    {:error, :rate_limited}

  {:error, :invoice_missing} ->
    {:cancel, :invoice_missing}

  {:error, {:validation_failed, _reason}} ->
    {:discard, :invalid_invoice_state}
end
```

This avoids wasting retries on failures that cannot succeed later.

## Example: Persist Progress For Multi-Step Work

Prefer durable state over implicit progress in memory:

```elixir
with {:ok, generating_run} <- ExportRun.mark_generating(run),
     {:ok, file} <- ExportGenerator.generate(generating_run),
     {:ok, upload} <- Storage.upload(file),
     {:ok, _completed_run} <- ExportRun.mark_completed(generating_run, upload) do
  :ok
else
  {:error, reason} ->
    ExportRun.mark_failed(run, reason)
end
```

The specific implementation can vary, but the important point is that the system can tell which step failed and what state the run is in now.

## Relationship To Other Guides

This guide works together with:

- `observability.md` for making failures diagnosable
- `post-release-verification.md` for checking behavior after rollout
- `elixir/background-jobs.md` for queue-backed execution details
- `elixir/external-system-integrations.md` for external boundary design
- `elixir/error-handling.md` for caller-facing failure contracts

Reliability and resilience are about making the workflow safe under real operating conditions.

The other guides describe adjacent parts of that work.

## Review Questions

- If this work runs twice, what prevents an incorrect second effect?
- For batch or fan-out work, is the intended failure strategy atomic, fail-fast, or accumulate-and-retry?
- If one step succeeds and the next fails, how will we know what already happened?
- Which failures are retryable, and which are permanent?
- What durable record exists if this workflow stalls, crashes, or is discarded?
- If the external outcome is ambiguous after a timeout, can we retry or reconcile safely?
- If this gets stuck in production, can someone resume, cancel, or compensate it intentionally?
