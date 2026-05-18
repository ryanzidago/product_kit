# Data Privacy And Compliance

## Rule

Collect, store, expose, log, and retain only the sensitive data the product actually needs.

Make deletion, access control, and third-party sharing explicit.

## Why

Privacy failures are expensive because they can create:

- legal and contractual risk
- irreversible data leakage through logs, exports, or third-party systems
- user trust damage that is hard to recover from

Sensitive data that was never collected or copied cannot leak later.

## Minimize Sensitive Data

Do not collect or store personal or sensitive data "just in case."

Before adding a field or payload, ask:

- do we need this data at all?
- do we need to persist it, or only process it briefly?
- do we need the full value, or only part of it?

Prefer the smallest useful data shape.

## Keep PII Out Of Logs, Errors, And Analytics

Do not log, emit, or forward personal or sensitive data unless there is a clear approved reason.

Be especially careful with:

- application logs
- error reporting
- analytics payloads
- debugging dumps
- exported files

PII leaked into logs is especially costly because it is hard to retract everywhere it was copied.

See [observability.md](observability.md) for general signal guidance.

## Retention Must Be Intentional

Do not keep sensitive data forever by default.

Know which data is:

- operational and still needed
- historical and intentionally retained
- temporary and safe to delete after processing

Retention should be a product and operational decision, not an accidental consequence of never cleaning anything up.

## Deletion And Anonymization Paths

If the system stores user-linked personal data, there should be a clear deletion or anonymization story.

Know which records:

- should be deleted fully
- should be anonymized
- must be retained for legitimate business, legal, or audit reasons

Do not design the data model so deletion becomes practically impossible.

## Control Access To Sensitive Data

Not every authenticated user or internal tool should see every sensitive field.

Access to sensitive data should be narrower when the domain justifies it.

Internal access still counts as access.

See [security-and-authorization.md](security-and-authorization.md) for broader trust-boundary guidance.

## Be Deliberate About Third-Party Sharing

Third-party systems should receive only the data they actually need.

This applies to:

- vendors
- analytics tools
- storage providers
- AI providers
- support tooling

Convenience is not enough reason to share full raw payloads.

## Derived Data Still Counts

Privacy obligations do not stop at the primary table.

Be careful with:

- analytics copies
- denormalized read models
- backups
- exports
- data warehouse loads
- audit snapshots

Derived systems can still leak or over-retain sensitive data.

## Relationship To Other Guides

This guide covers privacy and retention defaults.

Use adjacent guides for related concerns:

- [security-and-authorization.md](security-and-authorization.md) for trust boundaries and sensitive action protection
- [observability.md](observability.md) for diagnostics without leaking sensitive data
- [data-model-and-schema-design.md](data-model-and-schema-design.md) for minimizing unnecessary duplication of sensitive fields

## Review Questions

- Is PII excluded from logs, error messages, analytics payloads, and exports by default?
- Is there a deletion or anonymization path for this data when the product needs one?
- Are retention choices intentional, or are we keeping this data longer than necessary?
- Is access to sensitive data narrower than access to ordinary application data where appropriate?
- Are third-party services receiving only the minimum data they need?
- Does this change create new copies of sensitive data in derived systems such as exports, analytics, or backups?
- Has the change been checked against any relevant legal, contractual, or regulatory requirements?
