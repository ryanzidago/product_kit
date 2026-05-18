# Validation

## Rule

Validate on change, blur, or submit intentionally. Do not validate by habit.

## Validate On Change When

- feedback is cheap
- the user benefits from immediate guidance
- the field affects the rest of the form state

Good examples:

- required fields
- simple format checks
- dependent field visibility

## Validate On Blur When

- validating on every keystroke would feel noisy
- the check is expensive
- the field needs a more stable value before validation is useful

Good examples:

- long text inputs
- fields with expensive remote checks
- inputs where partial typing is often invalid but expected

## Validate On Submit Always

Submit is the authoritative validation boundary.

Even if a form validates on change or blur, the submit path must still validate everything correctly.

## Avoid

- showing a wall of errors on initial render
- expensive validations on every keypress with no debounce or clear benefit
- using flash for field-level validation errors

## Review Questions

- Why is this field validating on change, blur, or submit?
- Is the frequency proportional to the cost and user value?
- Does the submit path remain authoritative?
