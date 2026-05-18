# Bulk Operations

Bulk operations are powerful, but they bypass a lot of application-level behavior.

## Defaults

- `Repo.update_all/3` does not run changeset validations.
- `Repo.update_all/3` does not automatically maintain `updated_at`.
- `Repo.insert_all/3` does not automatically populate timestamps.
- Bulk operations also skip per-record application logic and association handling.

If the table has timestamps, set them explicitly.

## Prefer

```elixir
now = DateTime.utc_now(:second)

from(job in Job, as: :job)
|> where([job: job], job.state == :pending)
|> Repo.update_all(set: [state: :running, updated_at: now])
```

## Avoid

```elixir
from(job in Job, as: :job)
|> where([job: job], job.state == :pending)
|> Repo.update_all(set: [state: :running])
```

## `insert_all`

When using `insert_all`:

- set `inserted_at` and `updated_at` explicitly when the schema expects them
- think through `on_conflict` and `conflict_target` instead of using them casually
- batch large inserts instead of sending everything at once

## Use Bulk Ops Only When The Shape Fits

If the write depends on per-row validations, callbacks, or complex business logic, bulk APIs are usually the wrong tool.
