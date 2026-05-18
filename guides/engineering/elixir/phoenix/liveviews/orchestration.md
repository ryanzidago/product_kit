# Orchestration

## Rule

Keep the LiveView as an orchestration layer.

The LiveView should coordinate params, events, async work, and assigns. It should not own the whole feature.

## Belongs In Contexts

- business rules
- queries and persistence
- write operations
- cross-feature behaviour
- real authorization decisions

## Can Live In `...Live.Helpers`

Use a page-specific helper module when the logic is:

- pure
- specific to this page
- substantial enough to deserve tests

Good examples:

- mapping params into page state
- deriving table rows, sections, or badges
- shaping filter state
- deciding which empty state or CTA to show

## Keep Inline

Trivial one-off rendering glue can stay in the LiveView or component.

Do not create helper modules for tiny formatting functions that add indirection without clarity.

## Avoid

- large callback bodies full of business rules
- helper modules that quietly become a second context layer
- database access hidden inside LiveView-only helper modules

## Review Questions

- Would this logic still matter outside the LiveView?
- If yes, why is it not in a context?
- Is the LiveView orchestrating, or is it becoming the feature implementation?
