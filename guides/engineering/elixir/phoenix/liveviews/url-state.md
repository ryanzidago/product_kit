# URL State

## Rule

When page state changes what the user thinks they are looking at, prefer putting that state in the URL.

Drive that state from params and `handle_params/3`.

## Usually Belongs In The URL

- current tab
- filters
- search query
- sort order
- pagination
- selected item
- mode such as `new`, `edit`, or `compare`
- current wizard step

## Usually Does Not Belong In The URL

- dropdown open or closed
- hover state
- temporary loading flags
- transient validation errors
- tiny UI toggles that do not change the meaning of the page

## Use This Test

- If refresh should restore it, it probably belongs in the URL.
- If back and forward should traverse it, it probably belongs in the URL.
- If someone should be able to share the current view, it belongs in the URL.

## Why

URL-backed state makes the page:

- safer to refresh
- easier to reason about
- easier to test
- easier to share

## Review Questions

- Would a refresh preserve the meaningful state?
- Would back and forward behave naturally?
- Is there any state hidden in assigns that should really be a param?
