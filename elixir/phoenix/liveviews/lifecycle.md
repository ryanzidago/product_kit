# Lifecycle

## Rule

Keep each LiveView callback responsible for one kind of work.

## `mount/3`

Use `mount/3` for:

- session-derived defaults
- cheap initialization
- subscriptions or one-time setup

Do not put URL-driven page state here when `handle_params/3` should own it.

## `handle_params/3`

Use `handle_params/3` for:

- parsing params
- loading URL-driven page state
- reacting to patch-based navigation

If a refresh or patch should recreate the state, it likely belongs here.

## `handle_event/3`

Use `handle_event/3` for:

- user-triggered actions
- validation
- mutations
- kicking off async work

Keep it as orchestration, not as the home of all business logic.

## `handle_info/2`

Use `handle_info/2` for:

- async results
- PubSub messages
- internal messages

Do not route normal form or button actions through `handle_info/2`.

## Review Questions

- Is this work in the right callback?
- Would a refresh or patch recreate this state correctly?
- Is any callback taking on too many responsibilities?
