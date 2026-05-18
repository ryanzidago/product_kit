# Testing

## Rule

Test the behaviour that makes the LiveView important, not just that it renders.

## Important LiveViews Should Usually Prove

- authorization and access gating
- URL-driven state restoration through params
- important happy paths
- important validation failures
- loading, empty, and error states when they matter to the workflow
- navigation or redirect behaviour after mutations

## Form LiveViews

Form LiveViews should usually prove:

- validation behaviour
- successful submit
- failure on invalid input
- field or section behaviour that changes based on user input
- redirect or patch behaviour after submit
- recovery expectations for refresh or reconnect when the workflow is important

Do not stop at "submits successfully". Prove the workflow the user actually depends on.

## Index And Table LiveViews

Index or table LiveViews should usually prove:

- the expected records render
- empty state renders when there is no data
- search, filter, sort, or pagination state works through params when URL-driven
- row or action navigation goes to the right destination
- destructive or mutating actions update the page correctly
- unauthorized users cannot access the page or trigger protected actions

## Modal And Patch-Driven LiveViews

Modal flows should usually prove:

- opening the modal through the intended action
- modal state is restored correctly from params when patch-driven
- closing the modal returns to the right URL and state
- successful modal actions update or redirect correctly
- validation failures stay inside the modal flow and remain understandable

## Async LiveViews

Async LiveViews should usually prove:

- loading state
- failure state
- retry path when relevant
- stale results do not overwrite newer user intent
- URL or filter changes do not leave the page showing the wrong result

## Authorization Coverage

For important pages, prove both:

- the authorized user can complete the workflow
- the unauthorized user is blocked at the route and at the action boundary

Do not rely on "the button is hidden" as sufficient test coverage.

## Prefer Behaviour Over Markup

Prefer assertions about:

- what the user can do
- what the user sees next
- which state the page ends up in

Over assertions about:

- fragile wrapper markup
- incidental CSS structure
- every intermediate assign detail

## Avoid

- tests that only assert a page contains a generic heading
- brittle tests that overfit every markup detail instead of behaviour
- only testing the happy path for forms, async flows, or authorization
- skipping param-driven behaviour on pages that claim the URL is the source of truth

## Review Questions

- Does this test prove the behaviour the user depends on?
- Are the critical failures covered, not just the happy path?
- If the page is URL-driven, do tests prove that?
- If the page is a form, table, modal, or async flow, did we cover the checklist for that shape?
