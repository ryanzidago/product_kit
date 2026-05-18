# PR Splitting

## Rule

Split pull requests by coherent behavior slices, not by technical layer.

A good PR tells one reviewable story:

- what changed
- why that slice is valuable on its own
- how to verify it
- why it is the right foundation for the next slice

The target is the minimum valuable scope, not the smallest diff.

## What "Valuable" Means

A slice is valuable when it leaves the system in a more useful, more trustworthy, or more releasable state.

That value does not have to be a brand-new end-user feature.

Internal slices can still be valuable when they:

- prove a new data model under the existing product surface
- route old behavior through new invariants
- expose a new resource in a safe read-only way before writes exist
- reduce the risk of the next behavioral cutover

Prefer slices that produce a meaningful checkpoint over slices that only land plumbing.

## Prefer Behavior Over Layers

Do not default to splits like:

- backend PR
- frontend PR
- database PR
- cleanup PR

Those are ownership buckets, not review stories.

Prefer splits like:

- root-branch internalization
- branch discoverability
- actual branch forking
- consumer cleanup

Those are behavior or product slices. A reviewer can understand why the slice exists and what risk it is meant to retire.

## Tests For A Good Split

Use these questions when deciding whether a PR boundary is good:

1. Can the slice be explained in one sentence of user-visible or system-visible value?
2. Can a reviewer evaluate the important invariants without being distracted by unrelated churn?
3. Can the slice be verified on its own with focused tests or smoke checks?
4. Does merging it leave the branch in a meaningful state rather than a half-plumbed state?
5. Does it reduce risk or uncertainty for the next slice?

If the answer is "no" to most of these, the split is probably too mechanical.

## Preferred Shape For Stacked Work

When a feature needs multiple PRs, the default progression is:

1. Internalize the new primitive under the existing product surface.
2. Expose safe read paths or identifiers.
3. Land the real behavioral write path or cutover.
4. Follow with consumer adoption, examples, cleanup, or polish.

This is only a default, not a template. The real rule is that each PR should leave the system in a coherent state that is useful to review and safe to build on.

## Good First Slices Usually Look Like This

The first slice should usually prove the hard invariant work without forcing the full new UX or API surface immediately.

For example, prefer:

- add root branches and make existing conversation flows use them internally

over:

- add `branches` table and new foreign keys, but nothing actually uses them yet

The first version is valuable because it exercises the new model in real behavior.
The second version is mostly dormant plumbing.

## What To Avoid

- schema-only or plumbing-only PRs that do not change real behavior or enforce real invariants
- splitting tightly coupled behavior across PRs so neither PR is understandable on its own
- bundling examples, client churn, or cleanup into the main architecture-defining PR when they obscure the core review
- layer-based splits that force reviewers to reconstruct the product story from file ownership
- intermediate states that are difficult to test, reason about, or safely release

## When Layer-Based Splits Are Acceptable

Sometimes a layer-heavy split is still the right move, but only when the layer itself is the behavior boundary.

Examples:

- a standalone migration that must land before application code for operational reasons
- a pure documentation or examples PR after the contract is already settled
- an infrastructure or rollout PR whose only purpose is to change deployment mechanics

Even then, describe the PR by the behavior or operational outcome, not just the files it touches.

## How To Use This In Plans

In design plans:

- define the ideal end state first
- identify the architecture-defining decisions
- work backwards into V1, V2, V3
- describe rollout slices in behavior terms, not layer terms

In implementation plans:

- map tasks to reviewable checkpoints
- name stacked PRs after the behavior they land
- keep each PR focused on one review concern
- defer nearby cleanup unless it is required for that slice to make sense

## Example

For a conversation branching feature, a strong split is:

1. Root-branch internalization
2. Branch discoverability
3. Actual branching
4. Consumer cleanup

That is stronger than:

1. Database
2. Backend
3. API
4. Frontend

The better split gives each PR one real review question:

- are the new invariants correct?
- is the new resource model visible and understandable?
- does the new behavior work?
- have consumers caught up cleanly?
