# JavaScript Boundaries

## Rule

Prefer HEEx and Phoenix LiveView JS commands by default. Reach for a custom hook only when the browser or a third-party library truly requires it.

## Prefer LiveView JS For

- show and hide
- toggle classes
- transitions
- focus movement
- dispatching events
- navigation initiated from the client

## Use A Hook When You Need

- browser APIs that LiveView does not cover cleanly
- a third-party widget lifecycle
- DOM measurement or observers
- complex pointer, media, or canvas behaviour
- client state that must integrate tightly with external JS

## Avoid Custom JS For

- simple toggles
- basic modal behaviour
- navigation that should be a link
- class manipulation that LiveView can own

## Boundary Rule

If a behaviour is mostly server-driven UI state, keep it in LiveView.

If the behaviour is truly browser-native or library-driven, a hook is appropriate.

## Review Questions

- Does this really require a hook?
- Could this be expressed with LiveView JS instead?
- Will LiveView patches fight this custom JS?
