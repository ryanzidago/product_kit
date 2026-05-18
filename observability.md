# Observability

## Rule

If a feature matters, it should emit enough signals that someone can answer three questions quickly:

- is it being used?
- is it failing?
- is it getting slow?

Do not wait for production incidents to discover that the feature is opaque.

Add observability when building the feature, not only after something breaks.

## Why

Many production failures are hard to debug because the system does not expose the right information.

Common bad outcomes are:

- the feature fails, but the only signal is a vague exception with no context
- the feature is slow, but there is no latency signal tied to the workflow
- the feature is quietly falling back to a degraded path and nobody notices
- users stop completing the workflow, but the team only tracks crashes
- logs exist, but they do not include the identifiers needed to find one real execution

Observability is what makes production behavior explainable.

## Instrument The Feature, Not Every Helper

Do not try to make every private function observable.

Instrument the points where real behavior matters:

- request or job entry and exit
- important workflow steps
- external API boundaries
- background job execution
- expensive query or processing paths
- success, failure, retry, and fallback outcomes

Prefer a small number of high-signal events over a huge amount of debug noise.

## What Every Important Feature Should Expose

For important features, make sure you have:

- one usage signal
- one failure signal
- one latency signal
- enough context to correlate one real execution

That is the minimum useful baseline.

## Usage Signals

A usage signal answers:

- is the feature actually being exercised?
- how often?
- by which important slice, if that matters?

Examples:

- checkout started
- report generation requested
- payroll preview completed
- import job enqueued

Usage signals are often simple counters or feature-specific log events.

## Failure Signals

Failures should be visible in a tool meant for errors, not only buried in logs.

Useful failure signals include:

- unexpected exceptions
- retries above the normal baseline
- timeouts
- external API failures
- domain-specific failure outcomes that are important even when they are handled

Make the failure signal specific enough to answer:

- what failed?
- where?
- for which request, job, workflow, tenant, or record?

Do not flood error reporting with normal business outcomes that the system expects and handles.

## Latency Signals

If a feature is user-visible, queue-backed, or operationally important, measure how long it takes.

Useful latency signals include:

- request duration
- query duration
- external API duration
- job runtime
- end-to-end workflow duration

It is not enough to know that the request eventually succeeded.

You also need to know whether it became too slow, too expensive, or too noisy.

## Feature-Specific Signals

The best observability is often domain-specific.

Examples:

- number of rows processed
- number of records skipped
- fallback path used
- cache hit versus miss
- generated score range
- report completed with warnings

These signals are often more useful than generic CPU or memory charts when debugging a feature-level problem.

## Correlation And Context

A signal without context is much less useful.

Important logs, metrics, and errors should include enough stable context to trace a real execution, such as:

- request ID
- job ID
- tenant or account identifier when safe and appropriate
- feature or workflow name
- external correlation ID
- important path or step name

The goal is simple:

if a user, support engineer, or developer says "this run failed," you should have a way to find that run.

## Logs

Use logs for:

- workflow milestones
- unexpected branches
- retries and fallback paths
- useful summaries of what the system just did

Prefer structured logs with stable field names.

Avoid logs like:

- "entered function"
- "got here"
- giant dumps of raw structs with no curation

Logs should help you answer a question, not create more reading.

## Errors

Use an error-reporting system for:

- unhandled exceptions
- unexpected failures that deserve investigation
- boundary failures where extra context helps triage

When reporting an error, include context that explains the attempted operation, not only the stacktrace.

Do not turn every expected validation or business-rule failure into an error-reporting event.

## Metrics

Use metrics where trends and thresholds matter.

The most common useful metrics are:

- count
- duration
- failure count or rate
- queue depth or backlog

You do not need an enormous dashboard to start.

A few trusted feature-level metrics are better than dozens of unloved charts.

## Good Pattern

For each important feature, write down:

- what success looks like
- what failure looks like
- what slow looks like
- which signal will reveal each one

Example:

For "generate payroll preview":

- usage: preview started and preview completed
- failure: preview failed with error type
- latency: preview duration
- domain signal: number of employees or rows processed
- correlation: request ID and payroll run ID

That is enough to make the feature diagnosable.

## Relationship To Post-Release Verification

Observability and post-release verification are related but different.

- observability is about emitting the right signals
- post-release verification is about checking those signals after rollout

If observability is weak, post-release verification becomes guesswork.

## Avoid

- instrumenting everything equally
- adding logs with no identifiers or workflow context
- relying only on crash reporting and ignoring latency or degraded success
- shipping a feature with no clear way to measure whether it is healthy
- creating noisy signals nobody trusts
- dashboards with no owner and no concrete use

## Review Questions

- If this feature breaks, what signal will show it?
- If this feature gets slow, what signal will show it?
- If users stop completing the workflow, what signal will show it?
- Is this app already using telemetry or metrics, and if so, is this feature emitting the right signals there?
- Can we trace one real execution through logs, metrics, or errors?
- Are we adding high-signal observability, or just more noise?
