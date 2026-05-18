# Changesets

Changesets shape input and surface errors. They are not the source of truth for cross-row invariants.

## Defaults

- Keep `changeset/2` cheap, deterministic, and query-free.
- Use `changeset/2` for casting, required fields, normalization, and other validations that only depend on the given data.
- Put real invariants in the database, then reflect them back into the changeset with constraint helpers.
- If a rule truly needs a database read and cannot be expressed as a database constraint, make that read an explicit second step right before the write.
- Treat submit as the authoritative validation boundary. LiveView validate loops should stay lightweight.

## Choose The Right Boundary

Use the cheapest boundary that is still correct:

- single-row shape and value checks: changeset validations
- database-enforced row or cross-row invariants: constraints plus constraint mapping
- coordination across multiple writes or latest-state decisions: transaction and, when needed, locks

Do not stretch changesets into transaction orchestration. Do not stretch transactions to compensate for a missing database constraint.

## Reflect Database Constraints

If the database enforces something, the changeset should surface it cleanly too.

- Mirror `NOT NULL` columns with `validate_required/3` when handling external input.
- Mirror unique indexes with `unique_constraint/3`.
- Mirror foreign keys with `foreign_key_constraint/3`.
- Mirror check constraints with `check_constraint/3`.

This keeps the database as the final line of defence while still giving users field-level errors instead of generic failures.

When you add a new database constraint, update the relevant changeset in the same rollout plan so callers get shaped errors instead of raw database exceptions.

## Keep `changeset/2` Query-Free

`changeset/2` is often called from `phx-change`, tests, seeds, scripts, and background flows. Hiding I/O in the builder makes those call sites slower and less predictable.

Prefer `changeset/2` for:

- `cast/4`
- `validate_required/3`
- `validate_length/3`
- `validate_number/3`
- `validate_format/3`
- `validate_inclusion/3`
- pure normalization and derived field updates
- constraint mapping such as `unique_constraint/3`

Avoid putting these inside `changeset/2` by default:

- `Repo` queries
- `unsafe_validate_unique/4` on general-purpose builders
- remote API calls
- expensive preload-or-discover logic

If a UX flow truly benefits from an unsafe or expensive check during typing, make that an explicit choice in the caller. Do not silently make every `changeset/2` invocation pay for it.

## `unsafe_validate_unique/4`

`unsafe_validate_unique/4` is a UX helper, not an integrity mechanism.

- Use it only when early feedback is worth the extra query.
- Keep the real unique index in the database.
- Keep `unique_constraint/3` in the changeset too.
- Do not rely on `unsafe_validate_unique/4` for correctness. It can race.

Prefer it in narrow, intentional flows. Avoid it in reusable `changeset/2` builders that run in many contexts.

## If A Database Read Is Unavoidable

Before adding a query-backed validation, ask:

1. Should this be a database constraint instead?
2. Can the rule be checked from already-loaded data?
3. Does the rule only need to run on submit or write?

If the answer is still "yes, this needs a query", use this shape:

- keep `changeset/2` pure
- add a separate validator such as `validate_no_overlap/1`
- short-circuit when the changeset is already invalid
- guard on the relevant changed fields so the query only runs when it matters
- call the heavy validator in the context immediately before `Repo.insert/2` or `Repo.update/2`

When one field alone drives the expensive rule, `validate_change/3` is a good fit. When the rule depends on multiple fields, use an explicit change guard instead.

Name these validators plainly, for example:

- `validate_no_overlap/1`
- `validate_slug_availability/1`
- `validate_parent_state_allows_change/1`

The point is that reviewers can immediately see this is not the base changeset builder.

```elixir
defmodule MyApp.Bookings.Booking do
  use Ecto.Schema
  import Ecto.Changeset
  import Ecto.Query

  alias MyApp.Repo

  def changeset(%__MODULE__{} = booking, attrs) do
    booking
    |> cast(attrs, [:resource_id, :starts_at, :ends_at])
    |> validate_required([:resource_id, :starts_at, :ends_at])
    |> validate_light_rules()
    |> foreign_key_constraint(:resource_id)
  end

  def validate_no_overlap(%Ecto.Changeset{valid?: false} = changeset), do: changeset

  def validate_no_overlap(%Ecto.Changeset{} = changeset) do
    if Enum.any?([:resource_id, :starts_at, :ends_at], &Map.has_key?(changeset.changes, &1)) do
      if overlapping_period_exists?(changeset) do
        add_error(changeset, :starts_at, "overlaps an existing booking")
      else
        changeset
      end
    else
      changeset
    end
  end

  defp overlapping_period_exists?(changeset) do
    resource_id = get_field(changeset, :resource_id)
    starts_at = get_field(changeset, :starts_at)
    ends_at = get_field(changeset, :ends_at)
    current_id = get_field(changeset, :id)

    from(ep in __MODULE__)
    |> where([booking], booking.resource_id == ^resource_id)
    |> where([booking], booking.id != ^current_id)
    |> where([booking], booking.starts_at < ^ends_at and booking.ends_at > ^starts_at)
    |> Repo.exists?()
  end
end
```

```elixir
defmodule MyApp.Bookings do
  alias MyApp.Bookings.Booking
  alias MyApp.Repo

  def create(attrs) do
    %Booking{}
    |> Booking.changeset(attrs)
    |> Booking.validate_no_overlap()
    |> Repo.insert()
  end

  def update(booking, attrs) do
    booking
    |> Booking.changeset(attrs)
    |> Booking.validate_no_overlap()
    |> Repo.update()
  end
end
```

That gives you the intended behaviour:

- no query during LiveView `phx-change`
- no query when the changeset is already invalid
- no query when the watched fields did not change
- errors still attach to the field and surface normally to the user

This pattern improves cost and UX, but it is not a substitute for a real database constraint when the rule can race. In those cases the query-backed validator is only a better error-shaping layer.

## Single-Field Heavy Validation

When only one changed field drives the expensive rule, `validate_change/3` is a clean fit:

```elixir
def validate_slug_availability(%Ecto.Changeset{valid?: false} = changeset), do: changeset

def validate_slug_availability(%Ecto.Changeset{} = changeset) do
  validate_change(changeset, :slug, fn :slug, slug ->
    if Repo.exists?(from(post in Post, where: post.slug == ^slug)) do
      [slug: "is already taken"]
    else
      []
    end
  end)
end
```

This still needs a real unique index and `unique_constraint/3` if the slug must actually be unique.

## LiveView Boundary

LiveView form validation should usually call only `changeset/2`.

```elixir
def handle_event("validate", %{"booking" => params}, socket) do
  changeset =
    socket.assigns.booking
    |> Booking.changeset(params)
    |> Map.put(:action, :validate)

  {:noreply, assign(socket, :changeset, changeset)}
end
```

The submit path goes through the context, where the heavy validation is applied right before the write.

As a default boundary:

- LiveViews, controllers, and jobs build the base changeset
- contexts orchestrate expensive pre-write validation
- repos execute the write

The schema module can define heavy validators, but callers should not have to guess whether they need to run them. The write path owned by the context should make that explicit.

## `prepare_changes/2` Is Not The Default

`prepare_changes/2` is useful, but it solves a different problem.

- It only runs for valid changesets.
- It runs inside the same transaction as the repo operation.
- It is appropriate for atomic follow-up writes or the rare check that truly must share the transaction boundary.

Do not use `prepare_changes/2` as the default way to hide expensive validation inside `changeset/2`.

Reasons:

- it moves the extra query inside the write transaction
- it makes the write path heavier and keeps locks open longer
- it still does not replace a real database constraint for race safety
- it is easy to cargo-cult into places where an explicit pre-write validator is simpler

Use `prepare_changes/2` when atomicity is the point. Use an explicit second-step validator when cost control and clear call sites are the point.

## Bulk Operations

Bulk writes bypass changesets.

- `insert_all/3` does not run changeset validations
- `update_all/3` does not run changeset validations

If an invariant matters during bulk operations, it must be enforced elsewhere, usually in the database.

## Testing

Test each boundary at the boundary that actually owns the behavior.

- Test `changeset/2` as a pure function without database access.
- Test heavy validators with the database, because the whole point is the query-backed behavior.
- Test constraint-backed failures by asserting the write returns changeset errors through `unique_constraint/3`, `foreign_key_constraint/3`, or `check_constraint/3`.
- Test race-sensitive invariants at the database layer, not only through the pre-write validator.

Follow the general testing rule in `testing/abstract-vs-real-world.md`: keep small abstract tests for the contract, then add realistic tests for the important domain-shaped cases.

## Review Questions

- Does `changeset/2` stay cheap enough to call on every LiveView validate event?
- Is `unsafe_validate_unique/4` being used as a UX enhancement rather than as a correctness mechanism?
- Is every real invariant enforced in the database as well as surfaced in the changeset?
- If there is a query-backed validator, is there a clear reason it cannot be a database constraint?
- Does the expensive validator run only on the write path and only when the relevant fields changed?
- Does the context own the pre-write orchestration instead of scattering heavy validation decisions across callers?
