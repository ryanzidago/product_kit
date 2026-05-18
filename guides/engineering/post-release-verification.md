# Post-Release Verification

## Rule

Shipping is not done at deploy time.

For important changes, define a short post-release verification plan and actually check it after the feature reaches production.

This is not the same thing as automated testing.

Tests prove expected behavior before release.

Post-release verification checks whether the feature behaves correctly under real production conditions:

- real traffic
- real latency
- real data shape
- real failure modes
- real user behavior

## Why

Many production problems are not "the code crashes immediately."

They are problems like:

- requests succeed but are much slower than expected
- one path logs warnings or retries constantly
- errors spike only for a specific tenant, payload shape, or browser
- the feature technically works but users do not complete the workflow
- background work backs up after rollout

Automated tests and smoke checks help before release.

They do not replace watching the real system after rollout.

## When To Use It

Use post-release verification for changes that are important enough to carry meaningful production risk, such as:

- new user-facing workflows
- significant LiveView or API mutations
- business-logic changes with important downstream effects
- schema or query changes
- job, queue, or integration changes
- performance-sensitive paths
- auth, billing, or permission changes

Tiny copy changes or low-risk internal refactors usually do not need a formal verification window.

## Plan It Before Release

Do not wait until after deploy to decide what to watch.

Before release, answer:

- what should happen if this feature is healthy?
- what would failure look like?
- which signals would reveal that failure quickly?
- who will check those signals?
- when will they check them?

If the honest answer is "nobody will check," the rollout plan is incomplete.

## What To Watch

Pick a small set of concrete signals, not a vague instruction to "monitor production."

The usual categories are:

- logs
- error reporting
- latency and throughput
- background job health
- feature-specific success signals

### Logs

Look for:

- new warnings or error patterns
- repeated retries
- unexpected branches or fallback paths
- suspicious spikes in a feature-specific log line

Use the log system your team already has, such as Google Cloud Logging, application logs, or another central log console.

### Error Reporting

Watch error tools such as Sentry for:

- new exceptions after rollout
- rising volume on existing exceptions
- one environment, tenant, browser, or endpoint standing out
- recoverable errors happening far more often than expected

Do not stop at "there are no fatal crashes."

An error spike in one narrow path is still a bad release.

### Latency And Throughput

Check whether the feature is fast enough and stable enough in production.

Look for:

- request duration
- query duration
- job runtime
- queue depth
- timeouts
- retry volume

The important question is not only "does it work?"

It is also "does it still work at an acceptable speed and cost?"

### Feature-Specific Success Signals

The best verification signal is often domain-specific.

Examples:

- users complete the new workflow
- a generated report finishes in the expected time
- a new scoring rule produces plausible values
- the expected records are written after an event
- a rollout increases usage of the intended path rather than a fallback path

For some features, this matters more than raw error count.

## Keep The Window Short And Intentional

Post-release verification does not need to mean staring at dashboards forever.

For many changes, one short intentional check is enough:

- immediately after deploy
- after the first real traffic
- later the same day
- the next morning for slower workflows or batch jobs

Use a longer window only when the feature or rollout risk justifies it.

## Good Pattern

Prefer a release note or PR note that says something like:

- watch Sentry for new exceptions in `CheckoutLive`
- check request latency for `/reports/:id/run`
- verify the expected background job volume stays normal
- confirm at least one real production run completes successfully

That is a real verification plan.

"Monitor production" is not.

## Relationship To Testing

Do not treat post-release verification as a substitute for tests.

Do not treat tests as a substitute for post-release verification.

They solve different problems:

- tests reduce the chance of shipping a defect
- smoke checks help a human inspect realistic behavior before merge or release
- post-release verification checks whether the feature behaves correctly in the live system

Use all three when the change is important enough.

## Avoid

- assuming deploy success means feature success
- saying "we'll keep an eye on it" without naming signals
- watching only fatal errors and ignoring degraded performance
- checking logs without checking feature-specific outcomes
- making the verification plan so broad that nobody follows it
- depending on one person to remember manually every time

## Review Questions

- What real production signals would tell us this feature is healthy?
- What real production signals would tell us this rollout is going badly?
- Who is checking those signals, and when?
- Are we watching only crashes, or also latency, retries, and feature outcomes?
- If this feature fails in a subtle way, would our post-release verification notice?
