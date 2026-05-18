# Indexes

Indexes should follow real access patterns.

## Defaults

- Add indexes for columns used on hot `where`, `join`, and `order_by + limit` paths.
- Index foreign keys used for joins, existence checks, and delete or update cascades.
- Use composite indexes when the query shape depends on multiple columns together.
- If the query repeatedly targets a subset of rows, consider a partial index.
- Do not add speculative indexes without a real query path; every index adds write cost and storage cost.

## Good Heuristics

- index foreign keys
- index columns used to fetch "next N rows"
- index the same columns used together in recurring filters and sorts

## Example

```elixir
create index(:jobs, [:state, :inserted_at], concurrently: true)
```

For production index creation and lock-safety, follow [`safe-ecto-migrations.md`](./safe-ecto-migrations.md).

## Review Prompt

When a query is slow, ask:

- what columns does it filter on?
- what columns does it join on?
- what columns does it sort on before `limit`?
- does one index match that actual access pattern?
