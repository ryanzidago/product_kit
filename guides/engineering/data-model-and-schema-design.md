# Data Model And Schema Design

## Rule

Model the durable domain in the schema, not the current UI, API payload, or reporting convenience.

Make invalid states hard or impossible to represent.

Prefer schemas that say what the business facts are, who owns them, and which combinations are allowed.

## Why

The schema outlives pages, endpoints, jobs, and refactors.

A permissive or convenience-shaped schema creates long-term cost:

- invalid data gets stored and has to be cleaned up later
- domain rules drift into scattered application code
- UI decisions leak into tables that should have represented business facts
- every migration becomes harder because the data model never matched the domain cleanly

Getting the model right early is cheaper than correcting every row later.

## Model The Domain, Not The Current Surface

Tables should represent durable business concepts, relationships, and events.

Do not shape the schema primarily around:

- one screen
- one form payload
- one export format
- one API response
- one temporary workflow

Those surfaces can change quickly. The schema should survive them.

Prefer:

- `bookings`
- `booking_slots`
- `approval_runs`
- `invoice_line_items`

Over tables or columns shaped mainly around presentation convenience, such as:

- a wide table built only for one dashboard
- a single catch-all JSON blob for core business state
- one overloaded status column carrying unrelated concepts

If a read model or denormalized projection is useful for a UI or report, treat it as a derived shape, not the core domain model.

## Make Invalid State Hard To Represent

Ask whether the schema can represent data the business would consider invalid.

Prefer structure that prevents bad states where practical:

- `NOT NULL` when a value must exist
- foreign keys when one record depends on another
- unique constraints when identity or exclusivity matters
- check constraints when values have real bounds or allowed combinations

Do not rely on convention alone when the database can enforce the rule.

See [constraints-and-invariants.md](elixir/ecto/constraints-and-invariants.md) for Ecto-specific enforcement guidance.

## Nullable Should Be Intentional

Do not make columns nullable by default just because the shape is still evolving.

A nullable column should usually mean one of these:

- the value is genuinely optional in the domain
- the value is not known yet
- the value does not apply for this record type

If those meanings need different behavior, model them explicitly instead of collapsing them into one permissive `NULL`.

When a value should always exist after creation, prefer making that true in the schema instead of depending on every caller to remember it.

## Model State Explicitly

Use a boolean for a real independent binary fact.

Use an enum or status field for a mutually exclusive lifecycle state.

Prefer a timestamp when the transition time matters.

Examples:

- prefer `status: :draft | :published | :archived` over `is_draft`, `is_published`, and `is_archived`
- prefer `archived_at` over `is_archived` when the archive action has a real time and lifecycle meaning
- prefer `published_at` over `is_published` when publication time matters to the product

Multiple booleans often allow impossible combinations and usually indicate that the model wants one explicit state instead.

## Model Structured Values Deliberately

When a value carries real validation, querying, or lifecycle semantics, give it enough structure in the schema.

Examples:

- money often wants at least `amount` and `currency`
- a period often wants either `start_datetime` and `end_datetime` or a real database range type
- an address may want structured fields when the application needs validation, search, or country-specific formatting

Use a single freeform address field only when the application truly treats the address as opaque display text.

Use the simplest representation that matches the domain.

Two explicit fields such as `start_datetime` and `end_datetime` are often the clearest default.

Use a database range type when the range itself is the real concept and range-native queries or constraints are central to the model.

## Model Relationships Explicitly

Choose relationship shape deliberately:

- `1:1` when one record extends or owns exactly one other record
- `1:N` when one parent owns many children
- `M:N` when both sides are peers and the relationship itself may matter

If the relationship has its own data or lifecycle, model it as its own table instead of treating it as an invisible join.

Examples:

- memberships often deserve their own table because they carry role, state, and dates
- line items deserve their own table because quantity, price, and ordering belong to the relationship

Do not hide important domain relationships inside arrays or JSON blobs when they need integrity, querying, or independent lifecycle.

## Name Things With Domain Language

Table and column names should match the business language people use to discuss the system.

Prefer:

- `employment_contract`
- `billing_account`
- `approval_status`

Over vague or overloaded names like:

- `data`
- `info`
- `value`
- `type`
- `status` when multiple different status concepts are being collapsed into one field

The schema should make it obvious what a row means without opening a UI file or API serializer.

## Separate Source Of Truth From Derived Data

Persist the authoritative facts.

Derive convenience values, projections, summaries, and report-specific shapes intentionally.

Be careful when storing:

- duplicated counters
- denormalized labels
- cached status text
- summary fields that can drift from the underlying facts

These may still be worth storing, but the ownership must be clear:

- which field is authoritative
- which field is derived
- how drift is prevented or repaired

See [transactional-vs-analytical-databases.md](transactional-vs-analytical-databases.md) for broader source-of-truth guidance.

## Design For Change, Not Just Today

The right schema should make likely future changes survivable.

Prefer:

- additive tables and columns over overloaded meaning
- explicit relationship tables over encoded multi-meaning fields
- app-owned stable identifiers over display labels used as keys
- separate external IDs from internal IDs

Avoid:

- one column whose meaning changes by record type
- encoding multiple business concepts into one string field
- tables that only make sense if you already know one screen's layout
- generic escape hatches for core domain state when a real model is already clear

The goal is not to predict every future requirement.

The goal is to avoid choosing a schema that makes ordinary future changes unnecessarily painful.

## JSON And Flexible Fields

JSON or map-like fields can be useful for:

- low-value metadata
- sparse provider-specific payloads
- audit snapshots
- temporary ingestion buffers

Do not use them as the default home for core relational business facts that need:

- constraints
- joins
- filtering
- indexing
- lifecycle rules

Flexible fields are best when the flexibility is the real requirement, not when the model is merely under-specified.

## Soft Delete Is Not A Default

Use soft delete only when the domain actually needs recoverability, audit visibility, or delayed cleanup semantics.

Do not add soft delete preemptively to every table.

A soft-deleted table changes query behavior, uniqueness decisions, and lifecycle rules across the whole feature.

If deletion semantics matter, model them deliberately.

## Relationship To Other Guides

This guide describes what the schema should represent.

Use the narrower guides for implementation detail:

- [elixir/ecto/constraints-and-invariants.md](elixir/ecto/constraints-and-invariants.md) for database-backed truth
- [elixir/ecto/changesets.md](elixir/ecto/changesets.md) for validation boundaries
- [elixir/ecto/transactions.md](elixir/ecto/transactions.md) for coordinated writes and locks
- [architecture.md](architecture.md) for separating persistence from business logic and presentation

## Review Questions

- Does this schema model the domain, or is it mainly shaped by one UI or API today?
- Can the schema represent invalid states that the business would reject?
- Are nullable columns intentionally nullable, or just permissive by default?
- Are relationships modeled explicitly enough for the lifecycle and integrity they need?
- Are names aligned with domain language, or are they vague and overloaded?
- Which fields are authoritative facts, and which are derived convenience values?
- Will this model be painful to evolve when the domain grows or changes?
