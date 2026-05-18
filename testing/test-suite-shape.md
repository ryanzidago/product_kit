# Test Suite Shape

## Rule

Do not optimize the suite for the number of unit tests.

Optimize it for confidence, speed, and maintainability.

In an agentic workflow, this usually means:

- keep a base of small deterministic tests for pure logic and edge cases
- use targeted smoke checks for important business-logic and algorithm changes
- put strong emphasis on boundary integration tests
- use LiveView tests as the main UI confidence layer for Phoenix apps
- keep full browser end-to-end tests focused on a small set of critical journeys

The old "testing pyramid" is still directionally useful, but use it as a warning against an end-to-end-heavy ice cream cone, not as a rule that every codebase should maximize isolated unit-test count.

## Why

AI makes it cheaper to write tests.

It does not make slow, flaky, redundant, or low-signal tests cheaper to trust.

The expensive parts are still:

- waiting for broad tests to run
- debugging failures with weak localization
- maintaining brittle tests through framework and UI changes
- reviewing large amounts of generated test code that do not improve confidence

At the same time, agentic changes often cross boundaries:

- query shape plus database behavior
- LiveView event handling plus page state
- context logic plus background jobs
- serialization plus external API contracts

That means suites written for agentic code should usually put more weight on realistic integration coverage than a simplistic unit-heavy pyramid would suggest.

## Preferred Shape

Prefer this order of importance:

1. small deterministic tests for pure computation and sharp edge cases
2. targeted smoke checks for important business-logic flows
3. integration tests at real boundaries
4. LiveView tests for user-visible Phoenix behavior
5. a few browser end-to-end smoke tests

Do not read this as four separate silos.

Read it as a coverage strategy:

- push logic downward when it can be tested cheaply and clearly
- use a quick smoke pass when a change is broad enough that a human should see it run once
- keep boundary tests where real wiring matters
- prove important UI workflows without paying browser-level cost by default
- reserve full browser runs for the few flows that truly need them

## Small Deterministic Tests

Use small tests heavily for:

- calculation-heavy business rules
- parsing and coercion
- query-building helpers that do not need a real database
- policy and authorization logic that can be expressed without framework setup
- error translation and edge-case handling

These tests should:

- run fast
- fail locally
- name the exact rule being proven
- cover awkward edge cases that are expensive to discover through broader tests

Do not force framework orchestration or I/O through these tests just to make them look more "real."

## Smoke Checks

Use smoke checks as a fast sanity pass between narrow automated tests and heavier integration or browser testing.

For Elixir business logic, this often means running the changed code once in IEx with realistic inputs and inspecting the real output.

This is especially useful when an agent has:

- changed a long workflow across several modules
- generated many narrow tests but may still have missed a realistic scenario
- updated an algorithm, scoring rule, schedule calculation, or query builder
- produced code where the most important question is "does this behave plausibly on real input?"

Smoke checks are useful because they:

- catch obvious "looks wrong in practice" failures quickly
- give a human a direct read on the result shape
- help validate realistic input and output before investing in broader test setup
- complement generated tests that may overfit the implementation

But they are not a substitute for automated tests.

Treat them as:

- fast confidence checks during implementation and review
- a source for discovering missing automated test coverage
- a good fit for business logic and algorithmic changes

Do not treat them as durable regression protection on their own.

When a smoke check finds or proves an important case, capture that learning in an automated test at the right layer.

## Boundary Integration Tests

Put real emphasis here.

These are usually the highest-value tests in an agentic codebase because they prove that the code still works where mistakes are most common:

- Ecto queries against a real test database
- changes, transactions, and constraints at the repo boundary
- external API clients and serializers
- job enqueueing and job worker behavior
- controller, channel, or LiveView handoff boundaries

Prefer narrow integration tests over broad all-the-things tests.

A good integration test usually proves one real seam at a time:

- app code plus database
- app code plus queue
- app code plus external contract
- LiveView plus context plus persistence for one workflow

Do not skip these because an agent already wrote unit tests around helper functions.

## LiveView Testing

For Phoenix apps, LiveView tests should be the default UI-behavior layer.

Use them to prove:

- authorization and access gating
- param-driven page state
- form validation and submit flows
- patch and navigate behavior
- loading, empty, error, and success states
- important event-driven workflows
- modal and nested component behavior where the user flow matters

This is usually the best tradeoff for Phoenix:

- much higher confidence than isolated component helper tests
- much lower cost and flake risk than browser-driven end-to-end suites
- closer alignment with the actual user workflow than markup-only assertions

When deciding between another unit test and a LiveView test, prefer the LiveView test if the real risk is in orchestration, params, events, or state transitions.

## Browser End-To-End Tests

Use browser end-to-end tests intentionally, not by default.

They are worth paying for when they prove something lower layers do not prove well, such as:

- critical login or impersonation flows
- payment or checkout-style paths
- JS interop that LiveView tests cannot fully prove
- browser-specific behavior
- deploy-time smoke coverage for the most important journeys

Keep this layer small.

If a browser test fails and there is no lower-level test that helps localize the problem, add the missing lower-level test instead of only growing the browser suite.

## What This Means For Agentic Work

When AI writes or edits code, require tests that match the risk of the change.

Prefer:

- small tests when the change is genuinely local and computational
- smoke checks when the change is broad enough that you want to run realistic inputs once by hand
- integration tests when the change touches persistence, contracts, jobs, or framework seams
- LiveView tests when the change affects page workflows, params, or event handling

Do not accept a large pile of narrow tests as sufficient evidence for a workflow change.

Do not accept a smoke check as sufficient evidence for regression safety.

Do not ask for browser end-to-end coverage when a LiveView test or narrower integration test would prove the same thing more clearly.

The review question is not "did the agent add tests?"

The review question is "did the agent add the right test at the right layer?"

## Avoid

- treating unit-test count as a quality metric
- jumping straight to browser end-to-end tests for ordinary Phoenix workflows
- duplicating the same behavior at every layer
- mocking boundaries that are cheap and valuable to exercise for real
- broad integration tests that mix too many seams and fail opaquely
- markup-only LiveView tests that do not prove behavior
- treating a successful IEx run as a replacement for a real regression test

## Review Questions

- What is the real failure mode for this change: pure logic, boundary wiring, or user workflow?
- Is this behavior proven at the lowest layer that still gives real confidence?
- Should this change also get a quick smoke check with realistic inputs before review is done?
- Should this Phoenix workflow be proven with a LiveView test instead of a browser test?
- If this browser test is important, what lower-level test will help localize future failures?
- Did the added tests improve confidence, or just increase suite volume?
