# Redirects And Flash

## Rule

Use the navigation primitive that matches the scope of the change.

## Use `patch` Or `push_patch`

Use patching when the user stays in the same LiveView and only the URL-backed state changes.

Good examples:

- switching tabs
- opening a patch-driven modal
- changing filters or sort state

## Use `navigate` Or `push_navigate`

Use navigation when the user is moving to a different LiveView.

Good examples:

- leaving index for show or edit
- redirecting after a successful create or save
- moving to another workflow page

## Use `replace` Intentionally

Both link-based and socket-based LiveView navigation can replace the current browser history entry instead of pushing a new one.

Use `replace` when the URL change should not create an extra back-button stop.

Good examples:

- normalizing query params on first load
- redirecting from an alias route to a canonical route
- updating transient same-page state that would make the back button noisy if every change were recorded

Prefer the default push behavior when the user made a meaningful navigation choice they are likely to revisit.

Good examples:

- switching to a detail page from a list
- opening a substantial tab or mode the user may want to back out of
- moving through a workflow where back should return to the prior step

## Flash

Use flash for meaningful cross-view feedback and major action confirmation.

Do not use flash for:

- field validation errors
- small local UI state
- feedback that belongs inline beside the affected element

## Avoid

- patching when the user is actually leaving the page
- navigating when the user is only changing same-page state
- using `replace` for major navigation steps the user expects in browser history
- pushing a new history entry for every small automatic or transient state change
- using flash as a substitute for proper inline error handling

## Review Questions

- Is this state change still the same LiveView?
- Does the user need a different URL or a different page?
- Should this navigation push a new history entry or replace the current one?
- Should the feedback be inline, or is flash the right scope?
