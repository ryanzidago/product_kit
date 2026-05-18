# Performance

## Rule

Keep the socket state small, the render path cheap, and the event flow quiet.

## Avoid Oversized Assigns

- do not keep duplicate versions of the same data without a clear reason
- avoid storing large derived structures that can be shaped more cleanly
- keep only the state the page actually needs

## Avoid Expensive Render Work

- do not put heavy computation in `render/1`
- precompute once in callbacks or helper modules when needed
- avoid making every rerender rebuild large structures unnecessarily

## Avoid Noisy Events

- debounce or throttle where the user experience allows it
- do not fire expensive server work on every keypress without a good reason
- keep async boundaries clear for slow operations

## Use The Simpler Model

- use normal assigns for simple pages
- use streams only when they make large incremental collections simpler
- do not optimize into a harder-to-reason-about design too early

## Review Questions

- Is the socket carrying more state than the page needs?
- Is expensive work happening on every rerender?
- Are we triggering more events than the workflow really requires?
