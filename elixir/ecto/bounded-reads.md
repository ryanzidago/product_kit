# Bounded Reads

Queries should be deterministic and bounded by default.

## Defaults

- Do not ship unbounded `Repo.all/2` queries on request paths unless the table is intrinsically tiny and capped.
- When using `limit`, pair it with a deterministic `order_by`.
- Default to loading full schema structs in normal application code.
- Use narrower selects when you only need IDs, counts, aggregates, or another clearly intentional projection.
- For large or high-churn pagination, prefer keyset pagination over deep `OFFSET` pagination.
- Background jobs should also fetch work in bounded batches.

## Prefer

```elixir
from(event in Event, as: :event)
|> order_by([event: event], desc: event.inserted_at, desc: event.id)
|> limit(^page_size)
|> Repo.all()
```

## Avoid

```elixir
from(event in Event, as: :event)
|> Repo.all()
```

## Notes

- `limit` without `order_by` is not a stable definition of "first N rows".
- Loading full structs is often the simpler and safer default because the caller keeps a real schema value instead of an ambiguous partial projection.
- Partial schema selects can blur the line between "this field was not fetched" and "this field is actually `nil`".
- Use narrower projections when there is a clear performance or API-boundary reason, not as a blanket default.
- If the caller really needs the full dataset, that should be an explicit choice, not an accidental default.

If a query really is an intentional projection, make that shape obvious:

```elixir
from(event in Event, as: :event)
|> order_by([event: event], desc: event.inserted_at, desc: event.id)
|> limit(^page_size)
|> select([event: event], event.id)
|> Repo.all()
```
