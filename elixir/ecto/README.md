# Ecto Guides

This folder contains the default workspace conventions for application-side Ecto usage.

Each guide is self-contained on purpose:

- humans can read only the topic they need
- LLMs can retrieve a narrow file without dragging in unrelated guidance
- review comments can link to a specific convention instead of a long mixed document

## Guide Index

- [changesets.md](changesets.md) - Keep `changeset/2` cheap, reflect database constraints, and isolate unavoidable query-backed validation.
- [query-shape.md](query-shape.md) - Build queries with pipelines and named bindings so they stay stable as joins grow.
- [transactions.md](transactions.md) - Keep transactions thin and move non-race-sensitive reads outside them.
- [bounded-reads.md](bounded-reads.md) - Keep reads deterministic, bounded, and selective.
- [joins-and-preloads.md](joins-and-preloads.md) - Choose joins, preloads, and second queries intentionally.
- [indexes.md](indexes.md) - Add indexes for real access patterns, not guesses.
- [constraints-and-invariants.md](constraints-and-invariants.md) - Put real invariants in the database, not only in application code.
- [bulk-operations.md](bulk-operations.md) - Handle `update_all`, `insert_all`, and large batches explicitly.
- [fetching-and-cardinality.md](fetching-and-cardinality.md) - Choose `get`, `one`, `all`, and existence checks intentionally.
- [safe-ecto-migrations.md](safe-ecto-migrations.md) - Ship schema changes in phases and avoid lock-heavy migrations.

## Core Position

The broad shape across all guides is:

- pipelines over large inline `from` blocks
- cheap, query-free `changeset/2` builders
- named bindings over positional bindings
- thin transactions
- bounded reads
- intentional joins and preloads
- constraints as the final line of defence
- bulk writes with explicit timestamps and conflict behavior

Read the topic guide that matches the decision you are making. If multiple guides apply, prefer the stricter boundary.
