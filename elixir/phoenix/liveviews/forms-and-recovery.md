# Forms And Recovery

## Rule

When building a form, design for reconnect, refresh, and remount.

Prefer forms that can recover without losing meaningful progress.

## Default Phoenix Support

Phoenix LiveView automatically recovers form input values for forms that have:

- a stable `id`
- `phx-change`

For more complex or stateful flows, consider `phx-auto-recover`.

## Put Form Context In The URL

Keep form state in the URL wherever that is practical, especially:

- current step
- mode
- record id
- important surrounding filters or context

Do not rely on ephemeral assigns for meaningful progress.

## When URL State Is Not Enough

If the full draft cannot sensibly live in params, ask:

- which parts belong in params anyway?
- should this use a persisted draft?
- what should survive refresh versus only survive validation round-trips?

## Review Questions

- What happens if the socket disconnects mid-form?
- What happens on refresh?
- Does the form keep its place in the workflow?
- Is important progress recoverable rather than trapped in assigns?
