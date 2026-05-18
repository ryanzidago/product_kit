# Transactional vs Analytical Databases

## Rule

Use the transactional database as the authority for operational writes and source-of-truth state.

Use the analytical database as a derived system for reporting, analytics, exploration, denormalized read models, and large-scale historical or aggregate queries.

The boundary is about correctness, not just performance:

- transactional systems own authoritative current state
- transactional systems own writes that change business state
- analytical systems consume replicated, ingested, transformed, or backfilled data
- analytical systems may serve reads when stale or derived answers are acceptable
- analytical systems should not be treated as the authority for operational decisions

Examples such as Postgres and ClickHouse fit this pattern, but the rule is generic and does not depend on a particular product.

## When This Applies

This guide applies when a system uses both:

- a transactional store for application state and operational workflows
- an analytical store for reporting, event analysis, denormalized reads, or heavy aggregation

It is especially relevant when teams are deciding:

- where a user-facing write should go
- which system owns the current truth
- whether a read can tolerate replication lag
- whether an analytical read is safe to use in an operational flow
- how to handle CDC, ETL, materialization, and backfills without confusing derived data with authority

## Why This Boundary Matters

Systems with both transactional and analytical databases tend to fail in predictable ways:

- replication lag makes a recent write invisible in analytics
- denormalized tables drift from the normalized source state
- pre-aggregated tables answer summary questions but not exact current-state questions
- backfills temporarily rewrite or recompute historical results
- snapshot timing causes tables to disagree about "now"
- teams start using fast analytical reads as if they were authoritative

The problem is usually not that the analytical database is "wrong" in a general sense.

The problem is that it is often:

- slightly stale
- intentionally transformed
- missing transactional guarantees
- optimized for aggregate truth rather than exact operational truth

That makes it useful for reporting and dangerous for operational authority.

## Default Boundary

Prefer this default:

- operational writes go to the transactional database
- source-of-truth state lives in the transactional database
- analytical systems receive derived writes through replication, ingestion, ETL, materialization, or backfill
- operational reads that gate writes also come from the transactional database
- reporting and analytics reads come from the analytical database

If a feature changes business state, assume the transactional database owns it unless there is a very strong reason to do otherwise.

## What Transactional Systems Own

Transactional systems own user-facing and operational workflows where correctness matters more than broad analytical scan performance.

This usually includes:

- inserts, updates, and deletes that change business state
- transactions and multi-step workflows
- exact current-state reads
- authorization and entitlement checks
- quota, inventory, balance, and availability checks
- workflow transitions such as approve, cancel, settle, reserve, fulfill, or revoke
- uniqueness, foreign-key, and invariant enforcement
- reads that must reflect a recent write before the next step continues

When an operation needs read-after-write correctness, row-level constraints, or transactional visibility, it belongs in the transactional system.

## What Analytical Systems Own

Analytical systems own derived and read-heavy workloads where scale, scan speed, and aggregate analysis matter more than exact transactional freshness.

This usually includes:

- dashboards and reporting
- trend analysis
- BI and exploratory analysis
- product analytics and event analysis
- large scans across long time ranges
- wide joins across denormalized or historical datasets
- materialized summaries
- cohort and funnel analysis
- read models built for fast analytical filtering or grouping

Analytical systems are usually the right place for answering questions like:

- what happened over time
- how many
- which segments changed
- what patterns appeared across a large dataset

They are usually not the right place for deciding whether an operational action is valid right now.

## Read Guidance

Reads may safely come from the analytical system when:

- the read is informational rather than authoritative
- some lag is acceptable
- the result is aggregate, historical, or derived
- the read does not gate an operational write
- the UI can tolerate "as of" semantics
- a wrong or late answer would not cause an incorrect side effect

Examples:

- a reporting dashboard
- a usage summary card labeled with a recent refresh time
- cohort, retention, or funnel analysis
- an internal operations page that helps an operator investigate, but does not commit changes from analytical state alone

Reads must come from the transactional system when:

- the read determines whether a write may proceed
- the user expects exact current state after an action
- the result depends on transactional constraints or locks
- stale data could create duplicate, invalid, or conflicting writes
- the answer depends on normalized authoritative state rather than denormalized projections
- pre-aggregation or delayed ingestion could change the answer materially

Examples:

- checking whether a seat, room, or inventory unit is still available before booking
- checking whether an account is active before granting access
- checking whether a balance, quota, or entitlement still permits an action
- showing the result of a just-completed write where the user expects read-after-write correctness

## Write Guidance

The transactional database owns writes that change operational state.

This includes:

- user-initiated mutations
- background-job mutations that advance workflows
- status transitions
- balance and inventory changes
- permission and entitlement changes
- deduplicated command handling
- any write that relies on constraints, transactions, or exact current state

The analytical database may receive writes as derived data flows such as:

- replication or CDC from transactional tables
- raw event ingestion
- ETL or ELT loads
- denormalized projections
- materialized views or aggregate tables
- late-arriving corrections
- backfills and reprocessing jobs

Those writes are acceptable because they maintain analytical usefulness, not because they make the analytical system authoritative.

## Which Database Owns Source Of Truth

By default, the transactional database owns source-of-truth state for the application.

That means:

- if the transactional and analytical systems disagree, the transactional system wins
- operational code should treat analytical state as derived unless explicitly documented otherwise
- analytical schemas should be described as projections, aggregates, or historical stores, not as the canonical current state

This boundary should be explicit in architecture docs, code docs, and review discussions.

Do not let "near real-time" replication quietly turn into an assumption that the analytical database is also the authority.

## Correctness Vs Freshness

Freshness is not the same as correctness.

An analytical system can be only a few seconds behind and still be the wrong place to make an operational decision.

A read can be fresh-looking but still unsafe because:

- the relevant table has not received the latest replicated change
- related tables are from different snapshot points
- a denormalized row dropped information that matters for the decision
- an aggregate hides the exact record-level detail needed for correctness
- a backfill or reprocessing job is rewriting part of the dataset

Prefer this rule:

- analytical reads are acceptable when approximate or slightly stale truth is acceptable
- transactional reads are required when exact current truth determines the next operational step

## Acceptable Exceptions

Some exceptions are reasonable, but they should be documented explicitly.

Acceptable exceptions include:

- a product surface that is itself analytical and read-only
- observational event ingestion written directly to the analytical system when those events are not the authority for business state
- candidate selection or ranking computed analytically, followed by transactional revalidation before a write
- internal tooling that uses analytical reads for investigation while still confirming transactional truth before taking action
- systems where the true authority is an external operational source, while the local transactional store still remains the application's operational boundary

Even in these cases, operational writes and authoritative current-state decisions usually still belong in a transactional system.

## Failure Modes

Be explicit about the failure modes analytical systems introduce into operational thinking.

### Lag And Replication Delay

Recent writes may not appear yet.

This breaks:

- read-after-write user experiences
- operational decisions based on current status
- flows that assume the same user action is visible everywhere immediately

### Denormalization Drift

Analytical tables often flatten or reinterpret source data.

This breaks:

- decisions that depend on exact normalized relationships
- invariants that exist only in the transactional model
- assumptions that a projection row is equivalent to the source state

### Pre-Aggregation

Aggregate tables answer summary questions well but often cannot answer exact current-state questions safely.

This breaks:

- balance, quota, and availability checks
- decisions that need record-level detail
- logic that treats a summary as if it were a transactional ledger

### Backfills And Reprocessing

Analytical datasets are often rewritten, corrected, or recomputed.

This breaks:

- operational code that assumes historical rows are immutable
- workflows that depend on analytical results being stable during a rerun
- any logic that mistakes a corrected analytical view for the original operational event sequence

## Anti-Patterns

Avoid these patterns:

- using an analytical aggregate to decide whether a user may perform an operational action
- reading analytics to confirm a just-completed write that requires read-after-write correctness
- updating a denormalized analytical table and treating that update as the business state change
- enforcing permissions, entitlements, or account status from replicated analytical data
- treating a materialized count as exact current inventory or exact current balance
- driving workflow transitions from backfilled or recomputed analytical tables
- assuming low lag means the analytical database is safe for operational authority
- describing the warehouse or analytical cluster as "the real source of truth" for live operational state

## Practical Guidance

When in doubt:

- put the write in the transactional system
- keep authoritative current state in the transactional system
- replicate or ingest into the analytical system for reporting and analysis
- use analytical reads for insight
- use transactional reads for operational decisions

If a read might cause or justify a write, treat it as operational until proven otherwise.

If a read is only there to inform humans, summarize history, or explore patterns, the analytical system is often the better home.

## Example Boundary

Prefer:

- orders, payments, subscriptions, entitlements, and account status in a transactional database
- event streams, denormalized facts, reporting tables, and historical aggregates in an analytical database
- dashboards and usage reports served from analytics
- checkout, cancellation, approval, and entitlement enforcement served from the transactional system

Avoid:

- checking a replicated aggregate before charging a card
- granting access because a warehouse table says the account looks active
- updating an analytical summary row and treating the summary as the source state

## Review Questions

- Which database owns the write that changes business state?
- Which database is the source of truth for the current state this feature depends on?
- If the analytical pipeline is delayed by ten minutes, what breaks?
- If a backfill reruns tonight, could this read change tomorrow without a new operational event?
- Is this read informational, or does it gate a write or other side effect?
- Could denormalization, aggregation, or late-arriving data change the meaning of this answer?
- Does the user expect exact current status immediately after an action?
- If the transactional and analytical databases disagree, which one does the code trust?
- Is the analytical data clearly documented as derived, stale-tolerant, or "as of" data where needed?
