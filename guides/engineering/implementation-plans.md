# Implementation Plans

## Purpose

An implementation plan exists to turn an approved design into a safe execution sequence.

Its job is to make the delivery path clear:

- where to start
- what to change
- what order to do the work in
- how to verify each slice
- how to know when the work is complete

If a design plan answers "what are we building and why?", an implementation plan answers "how do we land it cleanly?"

## Rule

Write implementation plans as executable work orders derived from an already-decided design.

A good implementation plan makes the delivery path obvious:

- where the work starts
- which files or layers are affected
- what order to implement things in
- how each step is verified
- what “done” means at the end

Implementation plans are execution documents, not design debates.

They should stay focused on what to change and why that slice exists, with only enough operational detail to execute and verify it.

## Where New Plans Live

- Store new plans under `.artefacts/YYYY-MM-DD-topic/`.
- Name the file `<topic>-implementation-plan.md`.
- Keep it next to the related design plan.
- Link the design plan near the top of the document.

## When To Write One

Draft an implementation plan when the work is large enough that execution order matters:

- multi-file or multi-layer changes
- work that spans migrations, schemas, controllers, tests, and docs
- stacked PR or branch coordination; use [pr-splitting.md](pr-splitting.md) when deciding slice boundaries
- a feature that should be broken into safe vertical slices

Skip a full implementation plan when the change is small enough to execute directly without coordination risk.

## Standard Header

The existing plans converge on a compact header before the task list. Use the lines that matter:

- optional worker note when the plan will be executed mechanically
- `**Status:**` when the plan is stale, blocked, or superseded
- `**Prerequisite:**` when another plan or PR must land first
- `**Starting point:**` when branch or PR context matters
- `**Goal:**` one clear delivery outcome
- `**Architecture:**` the key execution model in a few sentences
- `**Tech Stack:**` only the dependencies and runtime details that affect implementation
- `**Design plan:**` link to the paired design plan
- `**References:**` official docs only when they materially help execution

## Common Structure

### 1. Overview section when the shape is non-trivial

Use sections like `## Data Model Overview` only when the implementer needs a concrete mental model before touching files.

### 2. `## File Structure`

Group work by type and action:

- `Create`
- `Update`
- `Replace`
- `Delete`
- `Keep`

This section should tell the reader where the plan will land before they read the tasks.

### 3. Task sections

Use one `## Task N: ...` section per coherent slice. Good task boundaries are:

- one schema plus migration
- one context boundary
- one plug or controller slice
- one endpoint family
- one cleanup or migration of old code

Inside each task:

- include a `**Files:**` list
- use checkbox steps with `- [ ]`
- start with a failing test when test-first execution is practical
- when showing test-first setup, include only test scaffolding, never test bodies
- include exact commands to run
- state the expected result after each important command
- add commit steps only when they genuinely help break the work into reviewable checkpoints

For test scaffolding, prefer placeholders like:

```elixir
test "<placeholder>" do
end
```

That is enough. Do not include assertions, setup, fixtures, or any other test body details in the plan. The plan should identify the contract or behavior being exercised, not pre-write the test.

### 4. Final verification

End with a `## Final Verification` section that runs the highest-signal checks for the repo or slice.

### 5. Completion criteria

End with `## Completion Criteria` describing what must be true when the plan is done.

## Writing Rules

- Break the work into small, ordered tasks that produce usable checkpoints.
- Keep each step imperative and verifiable.
- Name exact file paths.
- Name exact functions, schemas, routes, or commands when known.
- Carry forward important constraints from the design plan instead of assuming the implementer remembers them.
- Call out branch, PR, or migration-number dependencies explicitly.
- Prefer vertical slices over layer-by-layer batches when that keeps the system working incrementally. Name slices by behavior or review concern, not by file ownership.
- Focus on what to build, why the slice matters, and how to verify it. Do not overload the plan with large code samples.
- Keep test examples at pure scaffold level. No test bodies.

## What To Avoid

- one giant task with no checkpoints
- tasks that mix several unrelated concerns
- commands without an expected result
- hidden prerequisites
- re-litigating design decisions that should already be settled
- vague steps like “implement auth” or “update tests”
- any test body content beyond scaffold placeholders
- implementation snippets where a clear requirement would do

## Suggested Skeleton

```md
# <Topic> Implementation Plan

**Prerequisite:** ...
**Goal:** ...
**Architecture:** ...
**Tech Stack:** ...
**Design plan:** `.artefacts/.../<topic>-design-plan.md`

## File Structure

**Area:**

- Create: `...`
- Update: `...`

## Task 1: <First vertical slice>

**Files:**

- Create: `...`
- Update: `...`

- [ ] **Step 1: Write the failing test**
  Use placeholder scaffolding only:

  ```elixir
  test "<placeholder>" do
  end
  ```

- [ ] **Step 2: Run the test and verify the failure**
- [ ] **Step 3: Implement the slice**
- [ ] **Step 4: Re-run focused tests**

## Final Verification

- [ ] Run the repo-level checks that matter for this work.

## Completion Criteria

1. ...
2. ...
3. ...
```

## Drafting Checklist

- The linked design plan exists or the assumptions are frozen in writing.
- The implementer can tell where to start without opening the whole codebase first.
- Every task has a clear verification step.
- Final verification covers the full slice, not just focused unit tests.
- Completion criteria describe outcomes, not effort spent.
