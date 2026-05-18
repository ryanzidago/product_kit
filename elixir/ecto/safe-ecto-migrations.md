# Safe Ecto Migrations Learnings

This note extracts the main learnings from Fly's `safe-ecto-migrations` materials and translates them into team-friendly guidance.

Scope:

- The guidance is primarily Postgres-oriented.
- Some sections call out MySQL and MariaDB behavior, but the lock and migration advice is centered on Postgres.
- The main idea is not "never change schema in production"; it is "make schema changes in phases so reads and writes can continue safely."

## Core Principles

- Treat migration safety as a locking problem, not just a syntax problem.
- Split risky changes into multiple migrations instead of doing creation, validation, backfill, and cleanup in one step.
- Separate schema changes from data changes.
- Separate application rollout from destructive database changes.
- Err on the side of extra deployments over one big lock-heavy migration.
- Prefer migrations that are safe to re-run or resume.

## Ecto Migration Mechanics

By default, Ecto migrations rely on two protections:

- a migration lock so only one node runs a migration at a time
- a DDL transaction so partial failures roll back cleanly

These defaults are usually good and should stay on unless a migration explicitly needs different behavior.

Important exceptions:

- `CREATE INDEX CONCURRENTLY` cannot run inside a DDL transaction.
- bulk data backfills generally should not run inside the migration transaction

Useful switches:

```elixir
@disable_ddl_transaction true
@disable_migration_lock true
```

Use them only when the migration requires them. Disabling the migration lock means you now need another guarantee that only one node runs the migration.

## Operational Safeguards

- Prefer Postgres advisory locks for migrations that need concurrency-safe coordination without the default migration lock transaction.
- Add `lock_timeout` safeguards so unsafe migrations fail quickly instead of holding locks for too long.
- Consider `statement_timeout` as a broader safety net for runaway migration statements.
- If you use transaction callbacks like `after_begin/0` to set `lock_timeout`, remember they do not run when `@disable_ddl_transaction true` is set.
- Inspect likely lock behavior before shipping risky migrations.

Example custom migration wrapper:

```elixir
defmodule MyApp.Migration do
  defmacro __using__(_opts) do
    quote do
      use Ecto.Migration

      def after_begin do
        execute("SET LOCAL lock_timeout TO '5s'", "SET LOCAL lock_timeout TO '10s'")
      end
    end
  end
end
```

## Recipe Learnings

### Adding an index

- In Postgres, plain `create index(...)` blocks writes.
- Prefer `concurrently: true`.
- Keep concurrent index creation in its own migration.
- Do not mix unrelated schema changes into the same migration.

### Adding a foreign key

- Adding a validated foreign key can block writes on both tables.
- Add the reference with `validate: false`.
- Validate the constraint in a later migration with raw SQL.

Pattern:

```elixir
alter table("posts") do
  add :group_id, references("groups", validate: false)
end
```

Then later:

```elixir
execute "ALTER TABLE posts VALIDATE CONSTRAINT group_id_fkey", ""
```

### Adding a column with a default

- Adding a column with a default can rewrite the whole table.
- Add the column without the default first.
- Set the default in a later migration with raw SQL.
- Do not use `modify/3` just to set a default.
- If you need old rows to get the new value, do a backfill.

### Changing a column default

- Changing a default with `modify/3` can also trigger unnecessary type work and table rewrites.
- Use raw SQL `ALTER COLUMN ... SET DEFAULT` instead.
- Changing the default does not retroactively update already-written rows.

### Changing a column type

- Type changes are often unsafe because they can rewrite the table.
- Prefer a phased migration:
  1. add a new column
  2. dual-write in application code
  3. backfill old data
  4. move reads to the new column
  5. remove the old field from schemas
  6. drop the old column

### Removing a column

- Never remove a column while running application code still reads or writes it.
- First deploy code that stops referencing the column.
- Only then remove the column in a later migration.

### Renaming a column

- Prefer not to rename the database column at all.
- If the rename is mainly for application clarity, rename the schema field and point it at the old column with `source:`.
- If you truly need a database rename, treat it like a phased column replacement with dual write and backfill.

### Renaming a table

- Prefer renaming the schema/module instead of the underlying table.
- If the physical table must change, create the new table, dual-write, backfill, move reads, then drop the old table.

### Adding a check constraint

- Validating a new check constraint in one step can scan the whole table and block traffic.
- Create the constraint with `validate: false`.
- Validate it in a later migration.

### Setting `NOT NULL`

- Do not jump straight to `modify ..., null: false` on a populated table.
- First add an equivalent check constraint with `validate: false`.
- Backfill missing values.
- Validate the constraint later.
- On Postgres 12+, once the validated check proves the column has no nulls, you can set `NOT NULL` afterward without the usual full table scan.

### Adding JSON columns

- Prefer `:jsonb` over `:json` in Postgres.
- One concrete reason: `json` lacks an equality operator and can break existing `SELECT DISTINCT` queries.

### Squashing migrations

- When migration history gets long, use `mix ecto.dump` and `mix ecto.load` with `structure.sql`.
- This lets new environments fast-forward to the current schema state and then run only newer migrations.
- It is a practical way to reduce setup time without losing forward migration support.

## Backfill Learnings

Bulk data changes need their own discipline.

- Do not run one-off bulk fixes manually from a console.
- Get bulk data changes reviewed.
- Make them observable if they are a recurring operational pattern.
- Keep data migrations separate from schema migrations.
- Prefer running data migrations manually or via an explicit release command.

### Snapshot the schema

Do not use your application's live Ecto schemas inside old migrations.

Prefer one of:

- raw SQL against the table as it exists at migration time
- a tiny schema module defined inside the migration file that only models the fields needed for that migration

### Safe backfill rules

The Fly guide reduces safe backfilling to four ideas:

- run outside the migration transaction
- batch
- throttle
- build for resiliency

### Pagination strategy

- Avoid `LIMIT/OFFSET` for large backfills.
- Prefer keyset pagination.
- If you are running outside a transaction, you cannot rely on cursors.

### Deterministic backfills

If the update makes rows fall out of the query, you can repeatedly:

1. fetch the next batch with keyset pagination
2. update the batch
3. record the last processed ID
4. sleep briefly
5. continue until no rows remain

This works well for cases like "fill all rows where `approved IS NULL`".

### Arbitrary backfills

If updated rows are not easy to distinguish after mutation:

- create a real progress table, not an in-memory list
- populate it with the IDs that need work
- index it
- process small batches inside short transactions
- lock the target rows for update within each batch
- upsert the real-table changes
- delete completed IDs from the progress table
- throttle between batches
- drop the progress table at the end

The important idea is resumability: if the process fails halfway through, it should continue from saved progress instead of starting over.

## Suggested Team Defaults

- Default to phased migrations for anything beyond simple additive changes.
- Treat `validate: false` + later validation as the default pattern for constraints on populated tables.
- Treat backfills as operational work, not "just a migration body".
- Prefer raw SQL over `modify/3` when changing defaults or performing narrowly targeted DDL.
- Prefer schema-only renames over physical database renames when possible.
- Add safety rails like `lock_timeout` early so mistakes fail fast.

## Sources

- [fly-apps/safe-ecto-migrations](https://github.com/fly-apps/safe-ecto-migrations)
- [Safe Ecto Migrations](https://fly.io/phoenix-files/safe-ecto-migrations/)
