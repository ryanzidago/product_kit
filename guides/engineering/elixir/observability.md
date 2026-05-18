# Observability

## Rule

Emit observability at Elixir application boundaries and important workflow edges, not throughout every internal function.

Observe business logic at the level of meaningful domain operations:

- a payroll preview ran
- a schedule recomputation failed
- a scoring workflow used a fallback path
- a report job completed in 2.4 seconds

Do not try to observe every helper the workflow passed through.

## Why

When observability is added to every helper, the usual result is:

- noisy logs
- duplicated signals
- weak correlation across a real request or job
- instrumentation coupled to implementation details that change often
- core business logic that is harder to read and maintain

The production questions we usually care about are:

- did this workflow happen?
- did it succeed or fail?
- how long did it take?
- which path did it take?
- how much work did it do?

Those questions map better to workflow boundaries than to every private function call.

## Main Boundaries In Elixir Apps

The most common high-value boundaries are:

- Phoenix controller, channel, or LiveView entry points
- context or service functions that own a real business workflow
- Repo-heavy operations that materially affect feature behavior
- Oban worker `perform/1` boundaries
- external API facades and adapter calls
- async process boundaries where work hops from one process to another

Instrument those boundaries first.

If you still cannot explain real production behavior, then add narrower signals intentionally.

## What To Emit

Choose the signal type based on the question you need to answer:

- logs explain one concrete execution
- telemetry or metrics show counts, duration, trends, and thresholds
- error reporting captures faults that deserve investigation

Many important boundaries emit more than one of these, but they should not all carry the same detail.

## Observe Business Logic At Workflow Level

Business logic should absolutely be observable.

The question is where.

Prefer:

```elixir
def run_preview(payroll_run, opts \\ []) do
  started_at = System.monotonic_time()

  result = do_run_preview(payroll_run, opts)

  emit_preview_observability(payroll_run, result, started_at)

  result
end
```

Avoid:

```elixir
defp normalize_hours(hours) do
  Logger.info("entered normalize_hours")
  ...
end

defp apply_multiplier(hours, multiplier) do
  Logger.info("entered apply_multiplier")
  ...
end
```

The first shape tells you whether the domain operation succeeded, failed, slowed down, or used a degraded path.

The second shape mostly tells you that the code executed.

## Logs

Use logs at Elixir boundaries for:

- workflow start or finish when that helps debugging
- retries
- fallback paths
- degraded but recoverable behavior
- compact summaries of what the system just did

Prefer structured logs with stable keys.

Include useful identifiers such as:

- request ID
- job ID
- workflow name
- tenant or account ID when safe and appropriate
- important entity ID such as payroll run, report, or import ID

Avoid:

- logging every helper call
- free-form strings with no structured context
- giant dumps of assigns, params, or raw structs
- repeated logs for the same event at multiple layers

## Telemetry And Metrics

Use telemetry or metrics for signals you want to count, chart, alert on, or compare over time.

Common Elixir examples:

- request duration
- LiveView workflow duration
- Oban job runtime
- external API latency
- success and failure counts for a feature
- queue depth, retry count, or backlog
- rows processed or skipped for batch work

Telemetry is a better fit than ordinary logs when the team needs to answer:

- is this getting slower?
- is this happening more often?
- did failures spike after deploy?

## Error Reporting

Use error reporting for:

- unhandled exceptions
- unexpected failures at important boundaries
- integration failures that deserve investigation

Do not report every expected `{:error, reason}` as an operational fault.

Examples that usually should not become Sentry events by default:

- invalid user input
- business-rule denials
- expected not-found outcomes
- authorization failures that are normal and handled

Examples that often should:

- repeated unexpected adapter failures
- an exception inside an important workflow
- a job failure that indicates a real bug or outage

## Phoenix And LiveView

Phoenix entry points are often good observability boundaries because they already own request or page-level orchestration.

Prefer signals about:

- page or workflow entered
- important mutation succeeded or failed
- param-driven flow took a fallback path
- async load completed slowly or failed

Do not turn every `handle_event/3` into a pile of logging.

For LiveViews especially, prefer emitting observability for meaningful workflow outcomes instead of every click or assign update.

## Contexts And Service Modules

Contexts and workflow modules are often the best place to observe business logic.

This is where you usually know:

- what operation is happening
- which identifiers matter
- whether the result was success, failure, fallback, or warning
- how much work was done

If a context function is the stable public boundary for a real workflow, it is a strong candidate for observability.

## Repo And Database Work

Do not instrument every query by hand.

Do add observability when database behavior is a meaningful part of the feature:

- a report query is unusually expensive
- a batch operation processed a meaningful number of rows
- a transaction failed in an important workflow
- a migration-adjacent path is slower or noisier than expected

The point is to explain feature behavior, not narrate every Ecto call.

## Jobs And Async Work

Oban workers and async process boundaries should usually emit:

- start or enqueue context when useful
- success or failure outcome
- duration
- retry count or degraded state when relevant

Always include enough context to map the work back to the domain operation.

Job ID alone is rarely enough. Add the business identifier too when possible.

## External Integrations

Instrument the app-owned integration boundary, not only the HTTP client internals.

Useful signals include:

- provider name
- operation name
- success or failure outcome
- duration
- retry behavior
- external correlation ID when available

Do not log raw secrets or large sensitive payloads.

Keep the signal useful for debugging without turning production logs into data dumps.

## Correlation Across Processes

Elixir systems often cross process boundaries quickly:

- web request to context
- LiveView to Task
- context to Oban job
- job to external API

If you lose correlation context at those handoffs, production debugging gets much harder.

Preserve and propagate identifiers intentionally:

- request ID
- job ID
- workflow ID
- important entity IDs

The goal is that one real execution can still be traced after async hops.

## Good Pattern

For an important Elixir workflow, define one clear observability point near the public boundary.

At that point, emit enough information to answer:

- what operation ran?
- for which entity or account?
- how long did it take?
- did it succeed, fail, retry, or degrade?
- how much work did it do?

That is usually enough to make the workflow diagnosable without instrumenting every helper.

## Relationship To The Generic Guide

Use the delivery [observability.md](../delivery/observability.md) guide for the policy:

- what signals matter
- how to think about usage, failure, latency, and correlation

Use this guide for the Elixir-specific implementation boundary:

- where signals belong in Phoenix, contexts, jobs, and integrations
- how to avoid noisy or implementation-coupled instrumentation

## Avoid

- `Logger.info` scattered through pure helper functions
- telemetry events emitted from unstable low-level details when the workflow boundary would be clearer
- Sentry capture in every handled branch
- logs with no request, job, workflow, or entity context
- duplicate signals emitted from controller, context, and adapter for the exact same outcome
- turning business logic into instrumentation soup

## Review Questions

- What is the meaningful workflow boundary for this feature?
- Are we observing the domain operation, or just the functions it happened to call?
- Should this signal be a log, a metric, or an error event?
- Is this app already using telemetry, and if so, are we instrumenting this feature through the same telemetry shape instead of inventing a parallel pattern?
- Can one execution be traced across request, context, job, and integration boundaries?
- Did we add observability that explains production behavior, or just more noise?
