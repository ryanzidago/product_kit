# Security And Authorization

## Rule

Treat all external input as untrusted, verify identity explicitly, and enforce authorization at the boundary that owns the data or action.

UI checks are helpful, but they are not the final security boundary.

## Why

A single authorization gap can expose another tenant's data, allow an unsafe mutation, or leak sensitive information.

Security failures are expensive because they are often:

- hard to detect quickly
- trust-destroying
- difficult to reverse once data is exposed

## Authentication And Authorization Are Different

Keep these concerns separate:

- authentication answers who the actor is
- authorization answers what the actor may do

Do not treat a valid session, token, or logged-in user as proof that every action is allowed.

## Enforce Authorization At The Owning Boundary

Permission checks must survive:

- direct route access
- crafted params
- manual event firing
- alternate entrypoints such as jobs, APIs, or scripts

Prefer this layering:

- UI and LiveViews reflect permissions and gate navigation
- contexts, workflows, and data-access boundaries enforce the real rule
- scoped queries and writes ensure the actor can only reach allowed data

If a permission check only exists in the template, controller, or LiveView, it is incomplete.

See [elixir/phoenix/liveviews/authorization.md](elixir/phoenix/liveviews/authorization.md) for LiveView-specific guidance.

## Scope Data By Ownership And Tenant

In multi-tenant or ownership-sensitive systems, scoping is part of authorization.

Prefer queries and fetch functions that encode the ownership boundary directly.

Prefer:

- `get_invoice_for_account(invoice_id, account_id)`
- `list_reports_for_actor(actor)`

Over patterns that fetch by raw ID first and rely on every caller to remember a later permission check.

One missing scope can become a cross-tenant data leak.

## Treat Sensitive Actions More Strictly

Not every action carries the same risk.

Destructive or sensitive operations often need narrower scoping or stronger confirmation, such as:

- delete
- export
- impersonation
- billing changes
- permission changes
- access to sensitive personal data

Make these paths explicit.

Do not let them look like ordinary reads or low-risk updates.

## Validate And Sanitize Untrusted Input

Treat IDs, flags, role claims, filters, and payload fields from clients as untrusted input.

Validate shape and allowed values before the data reaches a dangerous boundary.

Sanitize data appropriately before rendering or forwarding it to another system.

Do not trust client input to declare:

- tenant ownership
- role membership
- record visibility
- permission to perform a destructive action

## Do Not Leak Secrets Or Sensitive Data

Do not log, serialize, or expose:

- secrets
- tokens
- raw authorization headers
- sensitive payloads unless there is a clear approved reason

Be careful with error messages too.

An error should not leak information that the caller was not allowed to know in the first place.

## Prefer Secure Defaults

When authorization is unclear, deny by default.

Prefer APIs and helper functions that require the auth context they need instead of making it optional or easy to forget.

Avoid broad helpers that can bypass ordinary scoping accidentally.

## Relationship To Other Guides

This guide covers the shared security boundary.

Use narrower guides for implementation detail:

- [elixir/phoenix/liveviews/authorization.md](elixir/phoenix/liveviews/authorization.md) for LiveView-specific enforcement
- [elixir/error-handling.md](elixir/error-handling.md) for shaping forbidden and not-found outcomes safely
- [observability.md](observability.md) for making sensitive failures diagnosable without leaking secrets

## Review Questions

- Is authorization checked at the owning boundary, not just the UI or controller?
- Can a user access or mutate another tenant's data through any path?
- Are inputs validated and sanitized before reaching the database or being rendered?
- Are secrets stored safely and kept out of logs, serialized payloads, and responses?
- Is authentication state verified on each request or action boundary where it matters?
- Are destructive or sensitive operations protected by narrower checks or confirmation?
- Has the change been considered from the perspective of a malicious authenticated user?
