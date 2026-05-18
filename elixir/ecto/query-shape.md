# Query Shape

Use a query shape that stays readable after the second and third join, not just on day one.

## Defaults

- Build Ecto queries with `|>` pipelines.
- Always name the root binding with `from(x in X, as: :x)`.
- Prefer named bindings in downstream clauses instead of positional bindings.
- Compose queries in steps so filters, joins, and selects can be added without rewriting earlier clauses.

## Why

Positional bindings are fragile once a query grows. Named bindings keep later edits local and make review easier.

## Prefer

```elixir
from(user in User, as: :user)
|> where([user: user], user.active == true)
|> order_by([user: user], asc: user.inserted_at)
```

## Avoid

```elixir
from(u in User,
  where: u.active == true,
  order_by: [asc: u.inserted_at]
)
```

## When A Query Grows

Prefer adding to the pipeline:

```elixir
from(user in User, as: :user)
|> join(:inner, [user: user], post in Post, as: :post, on: post.user_id == user.id)
|> where([post: post], post.published == true)
|> order_by([user: user], asc: user.inserted_at)
```

This is the default workspace query style unless a deeper project guide overrides it.
