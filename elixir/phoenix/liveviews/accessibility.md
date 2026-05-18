# Accessibility

## Rule

Dynamic UI must preserve semantics, focus, and keyboard access.

LiveView does not remove normal accessibility responsibilities. It makes some of them easier to forget.

## Semantics

- use real links for navigation
- use real buttons for actions
- keep heading structure meaningful
- prefer semantic HTML before adding ARIA

## Focus

- move focus intentionally when opening or closing modals
- do not let refresh-like updates silently dump focus in confusing places
- make sure keyboard users can continue the workflow after dynamic changes

## Keyboard

- every important action should be reachable by keyboard
- interactive elements must have visible focus states
- avoid click-only custom controls when native elements would work

## Dynamic Updates

- make sure loading, errors, and success are perceivable
- do not hide critical workflow changes only in subtle visual shifts

## Review Questions

- Can this workflow be completed with a keyboard?
- Will focus land somewhere sensible after dynamic UI changes?
- Are we using semantic elements before reaching for ARIA?
