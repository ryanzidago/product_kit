# Localization

## Rule

Separate temporal truth, translated copy, and locale-aware formatting.

Use:

- `DateTime` and explicit timezones for exact moments, scheduling, cutoffs, and business-day ownership
- `Gettext` for human-written copy that needs translation
- `CLDR` for locale-sensitive formatting of values such as dates, times, numbers, currencies, and units

These tools complement each other. They are not interchangeable.

## Why

Localization bugs usually come from mixing three different responsibilities:

- modeling what happened or when something is due
- choosing which words to show the user
- formatting values in the conventions the user expects

When those concerns get collapsed together, systems become fragile:

- business logic starts depending on presentation formatting
- UI strings start embedding timezone or formatting policy
- timestamps are stored as localized text instead of structured values
- locale and timezone get treated as if they were the same preference

Keep the data model, translation layer, and presentation formatting distinct.

Read this guide alongside [time-handling.md](time-handling.md). That guide covers temporal semantics in depth. This guide covers how time, translation, and locale formatting fit together at the product boundary.

## The Three Concerns

Use this framing first:

- time semantics answer "what instant or calendar boundary does this value mean?"
- translation answers "what words should this user see?"
- locale formatting answers "how should this value be rendered for this user?"

If the code is trying to answer more than one of those questions at once, the boundary is probably in the wrong place.

## Use `DateTime` And Timezones For Domain Truth

Use `DateTime` and explicit timezone handling when the system needs to model real temporal meaning:

- an invoice is due at a specific moment
- a job runs at `09:00` in an organisation timezone
- a reporting period closes at local midnight in the tenant timezone
- an SLA depends on the business day owned by a resource or organisation

This is not a translation problem and not a formatting problem.

The important questions here are:

- is this an exact instant, a local wall-clock value, or a date-only concept?
- which timezone owns the business rule?
- when should conversion from UTC to local time happen?

Default rules:

- store exact moments in UTC
- keep the owning timezone explicitly when local reconstruction matters later
- convert to local time where local meaning matters
- calculate business-day boundaries before formatting for display

Do not use `Gettext` or `CLDR` to compensate for weak time modeling. If the system does not know which timezone owns a cutoff, better translated strings will not fix it.

See [time-handling.md](time-handling.md) for the deeper rules around UTC storage, DST transitions, `DateTime` vs `NaiveDateTime`, interval boundaries, and business-day ownership.

## Use `Gettext` For Translated Copy

Use `Gettext` when the thing that varies by locale is human-written language:

- button labels
- page headings
- navigation labels
- email and notification copy
- validation and workflow error messages
- pluralized sentences

Examples:

- `"Save"`
- `"Shift starts at %{start_time}"`
- `"%{count} files uploaded"`

`Gettext` owns the words and sentence structure. It does not own how dates, numbers, or currencies are formatted.

Use `Gettext` to translate the surrounding message, then interpolate already-formatted values into that message.

Prefer:

```elixir
formatted_due_at = MyApp.Cldr.DateTime.to_string!(due_at, locale: locale)
gettext("Invoice due on %{due_at}", due_at: formatted_due_at)
```

Avoid:

```elixir
gettext("Invoice due on %{due_at}", due_at: due_at)
```

The raw `%DateTime{}` value is domain data, not user-facing text.

### What `Gettext` Should Not Own

Do not use `Gettext` to:

- format dates or times
- choose currency separators or decimal punctuation
- translate month names or territory names by hand
- decide which timezone a value should be shown in

Those are locale-formatting or time-modeling responsibilities, not translation responsibilities.

## Use `CLDR` For Locale-Aware Value Formatting

Use `CLDR` when the value is already known and the remaining question is how to render it for a locale.

Typical uses:

- formatting dates and times for display
- formatting numbers with locale-appropriate separators
- formatting currencies and currency symbols
- formatting percentages, units, and territories
- parsing user-entered localized values when the input format is locale-sensitive

Examples:

- `03/04/2026` vs `04/03/2026`
- `1,234.56` vs `1.234,56`
- `$1,234.56` vs `1.234,56 $`

`CLDR` owns presentation conventions. It should not decide business meaning.

Do not use `CLDR` to answer:

- when a billing period ends
- whether a token is expired
- what timezone owns a report cutoff
- whether two timestamps refer to the same instant

Those are `DateTime` and timezone questions.

### When `CLDR` Is Not Needed

Not every formatted date or number needs `CLDR`.

You usually do not need locale-aware formatting for:

- ISO 8601 timestamps in APIs
- logs and telemetry
- machine-oriented exports with a fixed contract
- internal debugging output

If the output is meant for systems rather than humans, prefer stable machine formats over locale-sensitive display formatting.

## Locale, Timezone, And Formatting Preferences Are Different Settings

Treat locale, timezone, and formatting policy as related but distinct concerns.

- locale answers language and many formatting conventions
- timezone answers local wall-clock interpretation and business-day ownership
- formatting preferences answer product-level choices such as hour cycle, date style, numbering style, or first day of week when those can be overridden independently

A user may want:

- English copy with a French timezone
- German number formatting with an American timezone
- UTC data entry with Portuguese copy
- English copy with `Europe/Paris` timezone and a `24-hour` clock
- French date formatting with a Monday-start week in one product and a Sunday-start week in another

Do not infer one from the other unless the product has a deliberate, documented policy and the fallback is only a convenience default.

Locale often gives a sensible default for:

- language
- date and number formatting conventions
- hour cycle
- first day of week

But those defaults are still defaults. They are not always the final product rule and not always the user's actual preference.

Prefer storing them separately when user preferences matter:

- `locale`
- `timezone`
- `hour_cycle` or equivalent display preference when supported
- `week_starts_on` or equivalent calendar preference when the product lets users or organisations change it

Do not collapse them into one field such as `region` if the application needs to make independent decisions about:

- language
- time display
- date formatting
- week ordering
- local time

### Sensible Defaults Are Fine

It is reasonable to derive initial defaults from locale, organisation settings, or region.

Examples:

- defaulting `en-GB` users to a `24-hour` clock
- defaulting `en-US` users to month-first date formatting
- defaulting a French organisation to a Monday-start business week

What matters is the boundary:

- a default is a starting assumption
- a stored preference or explicit business rule is the source of truth

Do not keep re-deriving product behavior from locale if the system already has a more specific setting.

## Compose Them At The Boundary

Most user-facing flows that "feel localized" actually involve all three layers in order:

1. start with structured domain values
2. resolve time in the correct timezone
3. format values for the user locale
4. translate the surrounding copy

Prefer:

```elixir
local_due_at = DateTime.shift_zone!(invoice.due_at, user.timezone)
formatted_due_at = MyApp.Cldr.DateTime.to_string!(local_due_at, locale: user.locale)

gettext("Invoice due on %{due_at}", due_at: formatted_due_at)
```

This keeps responsibilities clean:

- `invoice.due_at` is the real business timestamp
- `user.timezone` determines local wall-clock meaning
- `user.locale` determines display conventions
- `Gettext` provides the sentence

Avoid formatting too early in the domain layer and passing strings around as if they were business data.

## Keep Localization Out Of Persistence

Persist structured values, not user-facing localized strings.

Prefer storing:

- UTC timestamps
- `Date`
- numeric amounts in a machine-usable numeric type
- explicit locale or timezone preferences when needed

Avoid storing:

- `"March 23, 2026 at 9:00 AM"`
- `"1.234,56 EUR"`
- translated status labels such as `"En attente"`

Localized strings are presentation artifacts. They are hard to query, hard to compare, and often wrong once the locale, timezone, or copy changes.

Persist the underlying value. Reformat it at the boundary where it is shown.

## Keep Localization Policy Near The Presentation Layer

The closer code gets to the user boundary, the more localization concerns it should own.

Typical ownership:

- contexts and domain modules own temporal meaning and business rules
- web and email layers own user-visible translation and formatting
- export boundaries choose whether the export is machine-oriented or human-oriented

Do not push `Gettext` calls deep into domain code just because a workflow may eventually show the result to a user.

Prefer domain results like:

```elixir
{:error, :schedule_conflict}
```

Then translate at the web boundary:

```elixir
gettext("This schedule conflicts with an existing shift.")
```

This keeps the domain contract stable and lets different callers present the same failure appropriately.

## Common Combinations

Use this table when deciding which tool owns the problem:

| Problem | Primary tool | Why |
| --- | --- | --- |
| A job expires at one exact instant worldwide | `DateTime` and timezone rules | The core problem is temporal truth |
| A submit button should appear in Spanish | `Gettext` | The core problem is translated copy |
| A user should see `1.234,56` instead of `1,234.56` | `CLDR` | The core problem is locale-aware numeric formatting |
| A French user in `Europe/Paris` should see a due date in local time with French formatting | `DateTime` plus timezone, then `CLDR` | First resolve local time correctly, then format it |
| A translated sentence includes a localized date or amount | `Gettext` plus `CLDR` | Translate the sentence and interpolate a preformatted value |
| A recurring schedule runs every day at `09:00` in a tenant timezone | local schedule plus timezone rules | This is scheduling semantics, not translation |

When the answer is "all of them", the order still matters:

- first model the value correctly
- then localize the rendering
- then place it inside translated copy

## Common Mistakes

- treating locale and timezone as one preference
- treating locale as if it completely determines hour cycle, date ordering, or first day of week
- formatting dates or amounts in a context module and returning strings
- storing localized strings instead of structured values
- interpolating raw `%DateTime{}` or numeric values directly into translated copy
- translating month names, country names, or currency formats manually
- using locale-aware formatting for API payloads that should stay machine-readable
- assuming a translated UI automatically means the time logic is localized correctly

## Review Questions

- Is this problem about time semantics, translated copy, or locale-aware formatting?
- Are we using `DateTime` and explicit timezone rules for real temporal meaning?
- Are we using `Gettext` only for human-written copy?
- Are we using `CLDR` only for locale-sensitive value formatting or parsing?
- Are locale and timezone modeled as separate concerns?
- Are locale defaults being mistaken for explicit user or business preferences?
- Are we converting to the correct local timezone before formatting for display?
- Are we formatting values before interpolating them into translated strings?
- Are we persisting structured values instead of localized text?
- Is machine-oriented output using stable machine formats instead of locale-aware display formats?
