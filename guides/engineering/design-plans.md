# Design Plans

## Purpose

A design plan exists to decide what should be built before implementation starts.

Its job is to make the target shape clear:

- the problem being solved
- the recommended solution
- the boundary of the work
- the important decisions and tradeoffs
- what should explicitly wait

If a design plan is good, an implementer should not need to invent the architecture while coding.

## Rule

Write design plans to freeze the problem shape before execution starts.

A good design plan documents stable decisions:

- why this work exists now
- what boundary or system shape is changing
- what is in scope and out of scope
- which option is recommended and why
- what invariants, interfaces, and sequencing the implementation must respect

Design plans are decision documents, not task checklists.

In short:

- design plan = decide the shape
- implementation plan = execute the shape

## Reverse Design Planning

For feature work, start from the ideal end state rather than the easiest first implementation.

A good design plan should:

- define the best credible user and system experience for the feature
- identify which parts of that end state are architecture-defining
- work backwards into a staged rollout such as V1, V2, V3
- describe rollout slices in behavior terms when the work will land as stacked PRs; see [pr-splitting.md](pr-splitting.md)
- recommend the smallest V1 that keeps the path to later versions clean
- explicitly defer work that is useful but not architecture-defining

Do not shrink V1 by weakening the target architecture.
Shrink V1 by deferring work that does not constrain future versions.

## Where New Plans Live

- Store new plans under `.artefacts/YYYY-MM-DD-topic/`.
- Name the file `<topic>-design-plan.md`.
- Keep the paired implementation plan in the same folder when one exists.
- Use older examples under `.artefacts/` as references when they are still relevant.

## When To Write One

Draft a design plan when the work changes one of these:

- the domain model
- API shape or auth model
- system boundaries or ownership
- delivery sequencing across multiple slices
- a feature with multiple plausible approaches

Skip a design plan when the task is local, mechanically obvious, and does not require architectural decisions.

## What A Good Design Plan Answers

- What problem is being solved?
- What is true in the codebase today?
- What is the ideal end state for this feature?
- Which decisions are architecture-defining?
- What shape is recommended?
- What is the recommended V1?
- How does the work evolve from V1 to later versions?
- What alternatives are being rejected?
- What can safely be deferred?
- What must stay out of scope for now?
- What dependencies or prerequisites matter?
- What does success look like?

If a reader finishes the document and still cannot tell what to build or what not to build, the plan is too vague.

## Common Structure

Use the narrowest set of sections that makes the decision legible. From the existing artifacts, the most common sections are:

- `## Status` or `## Current State`
- `## Purpose` or `## Goal`
- `## References`
- `## Ideal End State`
- `## Architecture-Defining Decisions`
- `## Decisions` or `## Recommended Shape`
- `## Recommended V1`
- `## Reverse Rollout`
- focused design sections such as schema, auth flow, routing, API shape, or data model
- `## Deferred Work`
- `## Design Principles`
- `## Out of Scope`
- `## Relationship to Other Plans`
- `## Success Criteria`
- `## Immediate Next Step`

Use sections that match the problem. A schema-heavy plan should spend more space on entities and constraints. An API plan should spend more space on request and response shape, auth, and sequencing.

## Architecture-Defining vs Deferred

Treat a decision as architecture-defining when it materially affects one of these:

- domain boundaries or ownership
- external or internal interfaces
- data model shape or migration path
- core workflow sequencing
- extension points required for later versions

Usually not architecture-defining by themselves:

- cleanup
- naming consistency
- incidental deduplication
- local refactors
- polish that can be added later without changing boundaries

If a plan defers something important, explain why it is safe to defer.
If a plan includes something in V1, explain why it is required now.

## Writing Rules

- Lead with the problem and the recommended shape.
- Lead from the ideal end state toward V1, not from the easiest first slice upward.
- State concrete names when they are known: modules, schemas, tables, routes, plugs, contexts.
- Prefer decision tables when the plan contains several independent choices.
- Use examples, ERDs, or sample payloads only when they remove ambiguity.
- Keep rationale close to each decision instead of hiding it in a separate essay.
- Separate architecture-defining work from nearby cleanup and polish.
- Make exclusions explicit. The best plans say both what will be built and what will not.
- Explain why the recommended V1 preserves a clean path to later versions.
- If an older plan is stale, mark it clearly with `## Status` instead of silently editing history.
- Keep the document stable across implementation details. The plan should survive minor refactors.

## What To Avoid

- per-file implementation checklists
- commit scripts and branch instructions
- unresolved “maybe” language where a decision is actually needed
- broad goals with no boundary, no scope line, and no success definition
- pseudo-code that substitutes for making the actual design choice
- optimizing for the easiest V1 before defining the target architecture
- bundling non-essential cleanup into V1 without showing why it is required now

## Suggested Skeleton

```md
# <Topic> Design Plan

## Status

Optional when the plan supersedes older work or captures the current recommendation.

## Purpose

Explain the problem, the business or product reason, and why this work matters now.

## Current State

Describe the relevant state of the codebase today.

## Ideal End State

Describe the world-class target experience and system shape.

## Architecture-Defining Decisions

Capture the choices that must be right now to avoid later rewrites.

## Decisions

Capture the recommended shape and the rationale for each important choice.

## Recommended V1

Describe the smallest version worth shipping now.

## Reverse Rollout

Explain how the work evolves from V1 to later versions.

## <Boundary-Specific Sections>

Document the concrete interface, data model, flow, routing, or invariants.

## Deferred Work

List useful but non-architecture-defining work that should wait.

## Out of Scope

List what this plan does not cover.

## Relationship to Other Plans

Explain prerequisites, follow-up work, or sequencing.

## Success Criteria

Describe what must be true for the design to be considered complete.
```

## Drafting Checklist

- The title names the topic, not the meeting that produced it.
- The recommended shape is easy to find in the first screenful.
- The ideal end state and recommended V1 are both explicit.
- Architecture-defining decisions are separated from cleanup and polish.
- Scope boundaries are explicit.
- Cross-plan dependencies are named.
- The plan can be handed to someone else without a follow-up clarification meeting.
