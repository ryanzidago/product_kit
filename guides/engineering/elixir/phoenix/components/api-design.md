# API Design

## Rule

Design component APIs around semantic intent, not implementation leakage.

Call sites should describe what the UI means, not how the component happens to be styled internally.

## Prefer Semantic Assigns

Prefer assigns such as:

- `variant`
- `size`
- `label`
- `state`
- `disabled`
- `loading`

These age better than contracts built around raw class names or ad hoc booleans.

## Declare The Contract Explicitly

Use `attr` and `slot` declarations to make the API obvious and enforceable.

Prefer:

- required assigns only when the component truly cannot make sense without them
- slots for meaningful content areas
- `:global` attrs only when you intentionally want caller-supplied HTML attributes

## Let `class` Extend, Not Define

Caller-supplied classes are useful as an escape hatch, but they should extend a coherent component contract rather than replace it.

If every caller must rebuild the component with custom classes, the abstraction is weak.

## Avoid Leaky Contracts

Avoid component APIs that depend on callers knowing internals such as:

- exact wrapper structure
- fragile DOM ordering
- private CSS implementation details
- hidden coupling to event names or IDs

If consumers must understand internals to use the component safely, the API is not finished.

## Review Questions

- Does this call site read like intent or like implementation trivia?
- Are the important inputs expressed as explicit attrs or slots?
- Is `class` an extension point, or a sign that the contract is underspecified?
- Would another engineer know how to use this component correctly from its declaration alone?
