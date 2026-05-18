# Constraints And Invariants

If a bad state would matter in production, enforce it in the database.

## Defaults

- Use application validations for UX and error shaping.
- Use database constraints for truth.
- Prefer unique indexes, foreign keys, check constraints, and `NOT NULL` for real invariants.
- Reflect those constraints back into changesets so errors surface cleanly.

## Why

Application checks can race. Database constraints are the final line of defence.

## Examples

- A value that must be unique needs a unique index, not only a pre-insert lookup.
- A parent-child relationship needs a foreign key, not only a convention in code.
- A field with only valid ranges or states should use a check constraint when practical.

## Changeset Integration

If a constraint exists in the database, map it in the changeset too:

- `unique_constraint/3`
- `foreign_key_constraint/3`
- `check_constraint/3`

## If The Invariant Spans Multiple Writes

Some invariants still need transactions or locks. Use constraints where possible, then add transaction boundaries only for the parts that truly need coordinated reads and writes.
