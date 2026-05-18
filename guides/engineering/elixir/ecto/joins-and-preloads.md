# Joins And Preloads

Choose joins, preloads, and second queries intentionally. They solve different problems.

## Use A Join When

- the related table participates in filtering
- the related table participates in ordering
- the query is aggregating across tables
- the query needs row locking across the joined data

## Use A Preload When

- you already know the root rows you want
- you are loading associations for rendering or follow-up application logic
- the association should not affect which root rows are returned

## Defaults

- Do not join a table just to fetch associated data you could preload.
- Be careful with `1:N` joins because they multiply rows and often force `distinct`.
- If you only need to know whether a related row exists, consider `exists` or a subquery instead of `join + distinct`.
- A second intentional query is often clearer than one oversized join-heavy query.
- Keep join conditions complete and faithful to the real relationship.

## Prefer A Join For Filtering

```elixir
from(user in User, as: :user)
|> join(:inner, [user: user], post in Post, as: :post, on: post.user_id == user.id)
|> where([post: post], post.published == true)
```

## Prefer A Preload For Loading Related Data

```elixir
from(user in User, as: :user)
|> order_by([user: user], asc: user.inserted_at)
|> preload([:posts])
```

If a query starts accumulating joins only to avoid one extra query, stop and re-evaluate. That is a common route into both duplication bugs and N+1 confusion.
