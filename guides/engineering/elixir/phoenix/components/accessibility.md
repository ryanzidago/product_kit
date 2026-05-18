# Accessibility

## Rule

Reusable components own an accessibility contract, not just a visual contract.

If a component is meant to be reused, it should ship with accessible defaults rather than relying on every caller to remember them.

## Prefer Native Semantics

- use links for navigation
- use buttons for actions
- use real form controls before inventing custom ones
- prefer semantic HTML before adding ARIA

## Make States Perceivable

Reusable interactive components should account for relevant states such as:

- hover
- focus-visible
- disabled
- selected
- loading
- error

Loading and disabled states should not only look different. They should also communicate the change in behaviour clearly.

## Name Controls Clearly

- icon-only controls need accessible labels
- inputs need visible or programmatic labelling
- helper text and errors should stay associated with the relevant control

## Keyboard And Focus

- keyboard users must be able to operate the component
- focus states must remain visible
- opening, closing, or updating dynamic UI should leave focus somewhere sensible

## Review Questions

- If this component is reused widely, will it stay accessible by default?
- Does it rely on native semantics before reaching for ARIA?
- Are loading, disabled, and error states perceivable?
- Can a keyboard user complete the interaction?
