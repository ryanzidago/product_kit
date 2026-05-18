# Transactions

Transactions are for atomicity and locking boundaries, not for wrapping a whole flow by default.

Prefer `Repo.transact/1` and `Repo.transact/2` over `Repo.transaction/1` and `Repo.transaction/2`.

## Use A Transaction When

- multiple writes must succeed or fail together
- a read/write sequence depends on the latest locked state
- a database constraint alone is not enough and you need explicit row locking

## Keep Transactions Thin

- Read candidate IDs outside the transaction when staleness is acceptable.
- Compute expensive derived values outside the transaction when they do not depend on locked rows.
- Usually keep network calls, file I/O, sleeps, external API calls, and heavy CPU work outside the transaction.
- A small external side effect inside the transaction can be acceptable when you are deliberately biasing toward one simpler failure mode and you accept that the database and external system are still not jointly atomic.
- Avoid exploratory reads inside the transaction.

Every extra query inside a transaction increases contention and keeps locks open longer.

## Prefer Reads Outside When The Read Is Only Finding Work

```elixir
candidate_ids =
  from(job in Job, as: :job)
  |> where([job: job], job.state == :pending)
  |> select([job: job], job.id)
  |> limit(^batch_size)
  |> Repo.all()

now = DateTime.utc_now(:second)

Repo.transact(fn ->
  from(job in Job, as: :job)
  |> where([job: job], job.id in ^candidate_ids)
  |> Repo.update_all(set: [started_at: now, updated_at: now])
end)
```

This keeps the transaction focused on the write.

## Put The Read Inside When The Decision Can Race

Read inside the transaction when stale data can produce a bad write, for example:

- current state decides whether the write is allowed
- two workers could pick the same row
- you are incrementing, decrementing, or reserving something
- the rule depends on the latest value, not a slightly old snapshot

```elixir
Repo.transact(fn ->
  job =
    from(job in Job, as: :job)
    |> where([job: job], job.id == ^job_id)
    |> lock("FOR UPDATE")
    |> Repo.one!()

  if job.state != :pending do
    Repo.rollback(:not_pending)
  end

  job
  |> Ecto.Changeset.change(state: :running)
  |> Repo.update!()
end)
```

If the invariant can be enforced with a database constraint, prefer that as the final line of defence and handle the error explicitly.
