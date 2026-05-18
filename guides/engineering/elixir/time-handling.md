# Time Handling

## Rule

Default to UTC and use timezone-aware datetimes for business logic wherever possible.

Time bugs usually come from mixing three different concepts:

- exact moments in time
- local wall-clock values
- calendar dates

Treat them as different things. Do not use a weaker type just because it is easier to pass around.

## Core Defaults

- Default timezone: `UTC`
- Prefer `DateTime` for exact moments and business-owned temporal boundaries
- Prefer `NaiveDateTime` only for local wall-clock values that are intentionally not tied to one global moment yet
- Use `Date` only when the domain is irreducibly date-only and will not participate in timezone-owned comparison logic
- Use IANA timezone names like `"Etc/UTC"` or `"Europe/Athens"`, not ad hoc abbreviations like `"GMT"` or `"PST"`

## Choose The Right Type

### Use `DateTime` for exact moments

Use `DateTime` when the system needs to represent one real instant that should compare the same everywhere.

Examples:

- job scheduled to run at a specific moment
- message sent at
- audit event occurred at
- token expires at
- shift starts at

Store and compare these in UTC by default.

Also prefer `DateTime` when a business rule is calendar-shaped today but is likely to interact with:

- timezone ownership
- policy cutoffs
- timestamps from other parts of the system
- start or end boundaries
- ordering logic across mixed inputs

In those cases the datetime carries the comparison boundary explicitly instead of forcing each caller to invent one later.

### Use `NaiveDateTime` for local wall-clock values

Use `NaiveDateTime` when the value is intentionally local and cannot be understood without a separate timezone or context.

Examples:

- a report should cover `2026-03-21 09:00` to `2026-03-21 17:00` in a chosen timezone
- a store opens at a local datetime chosen by an operator before the timezone is finalized

Do not treat a `NaiveDateTime` as if it were globally meaningful on its own.

### Use `Date` only for irreducibly date-only concepts

Use `Date` when the domain is about the calendar day itself and there is no meaningful time-of-day or timezone comparison boundary to preserve.

Good fits:

- date of birth
- tax year start date
- a human-entered anniversary or commemorative date that is not compared against timestamps

Avoid `Date` when the value will later need to answer questions like:

- when does this day start or end for the owning business timezone?
- how does this compare to an existing timestamp?
- which side of a cutoff did this event land on?

Those are usually datetime questions, not date questions.

## Prefer Datetimes Over Dates

If a value could become ambiguous across timezones or will be compared against timestamps, prefer a datetime.

Prefer:

- `published_at`
- `starts_at`
- `processed_at`

Avoid:

- `published_on` when the system really means a timestamp
- `start_date` when the behavior depends on timezone or time-of-day
- `holiday_date` when the rule is really "from the start of that business day in the owning timezone"

`Date` is not more "simple" if the domain actually depends on timezone, ordering, expiry, or interval boundaries.

## Storage And Boundaries

- Store exact moments in UTC
- Convert to a local timezone only at the system boundary where local meaning matters: UI, reports, exports, business-day logic, or user-entered filters
- If local reconstruction matters later, store the timezone explicitly alongside the UTC timestamp
- Never assume the server timezone is the business timezone
- Never hard-code a regional timezone as a global default

## Clock Source

- Use wall-clock time for timestamps that describe when something happened
- Use monotonic time for measuring elapsed time, timeouts, retries, and performance
- Do not measure durations by subtracting wall-clock datetimes

Wall-clock time can move because of NTP sync, DST, or manual clock changes. Elapsed-time measurement should not.

## Serialization

- Serialize exact moments as ISO 8601 datetimes with an explicit timezone offset or `Z`
- Prefer UTC serialization by default for system-to-system boundaries
- Never send or persist naive datetime strings without an agreed timezone context
- If an API accepts local times, require the timezone or offset as a separate explicit field unless the owning timezone is unambiguous from the resource

Prefer:

```json
{"scheduled_at":"2026-03-21T09:30:00Z"}
```

Avoid:

```json
{"scheduled_at":"2026-03-21 09:30:00"}
```

## Naming

- Use `*_at` or `*_datetime` for exact moments and business-owned temporal boundaries
- Use `*_date` only for irreducibly date-only concepts
- Use `*_timezone` when the local timezone must be preserved or reconstructed later
- Use names that reveal the semantic boundary, not just the storage type

Prefer:

- `published_at`
- `starts_at`
- `effective_at`
- `start_timezone`

Avoid:

- `date` when the field is really a timestamp
- `effective_date` when callers will compare it against timestamps or business-day cutoffs
- `time` when the field is really a datetime
- `utc_time` if the value is actually a full timestamp

## Comparisons And Intervals

- Compare datetimes with datetime-aware functions, not raw operators
- Prefer closed-open intervals: start inclusive, end exclusive
- Avoid `23:59:59` end-of-day sentinels
- Represent "end of day" as the next day at midnight in the relevant timezone

Prefer:

```elixir
DateTime.compare(starts_at, ends_at) == :lt
```

```elixir
day_end = ~U[2026-03-22 00:00:00Z]
```

Avoid:

```elixir
starts_at < ends_at
```

```elixir
day_end = ~T[23:59:59]
```

## DST Transitions

Daylight saving transitions create two special cases for local wall-clock times:

- spring forward: some local times do not exist
- fall backward: some local times occur twice

Examples:

- `2026-03-29 01:30` may not exist in a timezone that jumps from `01:00` to `02:00`
- `2026-10-25 01:30` may be ambiguous in a timezone that repeats that hour

Rules:

- Do not silently guess when converting a local `NaiveDateTime` plus timezone into an exact moment
- For nonexistent local times, prefer rejecting the input with a clear validation error
- For ambiguous local times, require an explicit choice unless the domain has a clear default policy
- If the domain does have a default policy, document it and apply it consistently everywhere
- Test both spring-forward and fall-back cases for any timezone-aware scheduling or interval logic

For recurring local schedules like "run every day at 09:00 local time", keep the schedule definition in local time and resolve each occurrence against the timezone at execution time. Do not precompute one fixed UTC time and assume it stays correct across DST changes.

## Business-Day Ownership

- Be explicit about which timezone owns each business-day calculation
- Define whether "today", "this week", payroll cutoffs, reporting windows, and SLAs are based on UTC, user timezone, organisation timezone, or resource timezone
- If the business day starts at a non-midnight hour, document that rule and use it consistently in queries, reporting, and validation

Two systems can both be "correct" and still disagree if one computes the day in UTC and the other computes it in a local business timezone. That is a design decision, not just a formatting issue.

## User-Facing Time

- Show users local time, not raw UTC
- Compute "today", "this week", and "end of month" in the relevant business or user timezone
- Convert first, then take the `Date`
- Be explicit about which timezone owns a calculation

Avoid using `Date.utc_today()` for user-facing behavior unless the domain is explicitly UTC-based.

## Precision

- Pick one default precision for new exact-moment fields and use it consistently
- Prefer UTC microsecond precision for new persisted timestamps unless there is a concrete reason not to
- Be careful when comparing values from mixed precisions such as seconds vs microseconds
- Do not rely on string formatting or implicit truncation to decide equality

## Testing

- Freeze or inject the clock when the behavior depends on "now"
- Test around midnight boundaries in the owning timezone
- Test month-end, year-end, and leap-day boundaries when the feature depends on calendar logic
- Test DST transitions for timezone-aware scheduling and interval features
- Include at least one case where local date and UTC date differ

## Review Questions

- Is this value an exact moment, a local wall-clock value, or a calendar date?
- If two users in different timezones read this value, should they agree on the same instant?
- Are we using `Date` where the domain will eventually need datetime boundaries anyway?
- Are interval bounds closed-open?
- Are we converting to local time only where local meaning is actually needed?
- Is the timezone explicit, or are we relying on an ambient default?
- Have we defined what happens for nonexistent and ambiguous local times around DST transitions?
- Are we using the right clock source for timestamps vs elapsed-time measurement?
- Are API and job payloads serialized with explicit timezone information?
- Have we stated which timezone owns the business-day calculation?
