# Streams

## Rule

Use streams when they simplify a long-lived collection. Do not use them by default.

Streams are most helpful when the page incrementally inserts, deletes, or updates individual items in a large list.

## Good Fits

- timelines
- activity feeds
- chat-style lists
- tables that receive targeted row updates
- collections where replacing the whole list would be noisy or expensive

## Bad Fits

- small lists that are easy to reason about normally
- pages where the whole collection is constantly re-filtered or re-sorted
- flows where the source of truth becomes unclear once data is in the stream
- cases where a plain assign is easier to test and understand

## Heuristic

If the stream makes the data flow harder to explain, do not use it.

If the page mostly thinks in terms of "replace the whole list", a normal assign is often the clearer choice.

## Review Questions

- Is this collection truly incremental?
- Does the stream simplify rerenders, or just add indirection?
- Can another developer quickly explain where the source of truth lives?
