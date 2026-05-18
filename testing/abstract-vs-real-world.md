# Abstract Vs Real-World Testing

Use both of these test shapes when the behavior matters:

- **Abstract behavior tests** verify the core contract in the smallest, clearest form.
- **Real-world example tests** verify that the same contract still holds against realistic data shapes, names, and values.

Do not force a choice between them. We want both.

## Why

Abstract tests make the rules obvious and keep failures easy to diagnose.

Real-world tests catch gaps that abstract fixtures often miss:

- misleading field names
- realistic IDs, timestamps, and enums
- production-like table shapes
- behavior that is technically correct in a toy example but confusing or broken in practice

## Recommended Pattern

For important logic, keep the test suite layered:

1. Start with **abstract behavior tests**.
2. Add **real-world example tests** for the most important production-shaped scenarios.

Use abstract tests to prove the generic rule.

Use real-world tests to prove that the rule survives contact with actual domain data.

## What Counts As Abstract Behavior Testing

These tests should:

- use the smallest fixture that makes the rule clear
- focus on one behavior at a time
- assert the exact contract, not a broad end-to-end story
- stay easy to read without needing business context

Examples:

- compiler rejects unknown fields
- query builder emits `GROUP BY` for selected dimensions
- filter coercion turns `"true"` into `true`
- time-grained dimensions emit the expected alias

## What Counts As Real-World Example Testing

These tests should:

- use realistic field names, values, and table shapes
- mirror an actual workflow, report, or query people care about
- preserve the concrete names that make the scenario recognizable
- assert the important output, not just that the function returns `:ok`

Examples:

- metrics table with `metric_name`, `timestamp`, `value`, `type`, `is_current`, and `org_unit_id`
- payroll or scheduling data with realistic organisation-scoped identifiers
- a real dashboard query shape that product or support would actually discuss

## Selection Rule

Add a real-world example test when at least one of these is true:

- the behavior is central to the feature
- the domain vocabulary carries meaning that a toy fixture would hide
- the bug risk comes from realistic combinations of fields or values
- the example is likely to appear in docs, PRs, support debugging, or product discussion

## Balance

Do not replace abstract tests with only realistic tests.

Do not hide important product behavior behind only toy fixtures.

Prefer this combination:

- a few small abstract tests for rule coverage
- one or more realistic tests for important production-shaped examples

## Good Outcome

A reader should be able to answer both questions from the test file:

1. What is the rule?
2. What does that rule look like on real data?
