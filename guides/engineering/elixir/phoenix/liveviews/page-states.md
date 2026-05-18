# Page States

## Rule

Every important LiveView should define its loading, empty, error, and success states intentionally.

Do not let these states emerge accidentally from missing assigns or partial templates.

## Loading

- show a clear loading state for initial load and meaningful follow-up work
- prefer preserving stale data while the next result loads when that keeps context clear
- do not blank the whole page unless there is no sensible partial UI to keep visible

## Empty

- make empty states explicit
- explain why the page is empty
- include the next sensible action when appropriate

## Error

- make failure visible near the affected area when possible
- offer retry when the action can be retried
- do not silently fall back to stale or partial data without telling the user

## Success

- show clear confirmation for meaningful mutations
- prefer inline success states or flash only when it helps the user understand what changed

## Review Questions

- What does the user see while data is loading?
- What does the user see when there is no data?
- What does the user see when the action fails?
- Is the success state visible and understandable?
