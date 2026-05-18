# Fetching And Cardinality

Choose the repo function that matches the number of rows you expect.

## Defaults

- Use `Repo.get/2` for primary-key lookup when missing rows are acceptable.
- Use `Repo.get!/2` when absence is truly exceptional.
- Use `Repo.one/2` when zero or one row is expected.
- Use `Repo.one!/2` when exactly one row must exist.
- Avoid `Repo.all/2 |> List.first()` or similar cardinality-by-accident patterns.

## Existence Checks

If the question is "does a row exist?", prefer an existence check over loading rows you do not need.

## Default To Full Schemas

Default to full schema structs in normal application code.

This keeps the return shape obvious and avoids ambiguity around partial schema loads where an unfetched field can look the same as a real `nil`.

## Use Narrower Selects Intentionally

Use narrower selects when the query is naturally returning:

- IDs
- counts
- aggregates
- existence checks
- a clearly intentional projection for a boundary or reporting use case

Do not partially select a schema struct just because "fetch fewer columns" sounds cleaner in theory. In most application code, the full schema is the safer default.

Prefer:

```elixir
from(user in User, as: :user)
|> where([user: user], user.active == true)
|> Repo.all()
```

Prefer a narrow projection only when that is the real shape you want:

```elixir
from(user in User, as: :user)
|> where([user: user], user.active == true)
|> select([user: user], user.id)
|> Repo.all()
```

## Counts

If the caller needs a count, perform a count. Do not load rows just to count them in Elixir.
