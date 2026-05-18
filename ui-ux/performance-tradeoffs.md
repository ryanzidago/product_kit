# UI Performance Tradeoffs

## Rule

When two UI paths serve the user goal similarly well, prefer the one that costs the system less.

Use UI structure, defaults, and navigation to avoid unnecessary heavy work. Make expensive pages, queries, and computations intentional rather than automatic.

## Why

Small UI decisions can create large backend cost differences:

- the default landing page may trigger much more database work than a simpler overview
- an automatically loaded tab may run queries many users never need
- a detailed first screen may force joins, preloads, or aggregations before the user has chosen a narrower path
- a lightweight entry page can often preserve task success while reducing load, latency, and contention

This is not about making the UI worse to save resources.

It is about noticing when a small change in flow, default state, or information density can preserve the user outcome while avoiding expensive work across many requests.

## Prefer Low-Cost Defaults

If one page is substantially heavier than another, do not make the heavy page the default unless it is clearly the user's primary starting point.

Prefer:

- an overview page before a heavy detail or analytics page
- a summary card before a full table with broad joins and filters
- a lightweight home screen that helps users choose where to go next
- default routes, redirects, and first-load states that avoid expensive reads when the lighter path is still useful

For example, if users can start from either:

- a lightweight queue or summary page
- a heavy page that loads many records, derived metrics, and broad filters

Prefer sending them to the queue or summary page first, then let the heavier page be a deliberate next step.

## Make Expensive States Intentional

Do not automatically load the most expensive tab, panel, or report just because it exists.

Prefer requiring a meaningful user choice before expensive work begins:

- select a date range before loading a large report
- choose a record before loading related history
- expand a section before fetching deep supporting data
- navigate to the analytics screen instead of embedding the analytics query into the default operational page

This keeps the default path cheap and aligns cost with user intent.

## Prefer Progressive Disclosure

Show enough information to help the user decide what they need next, then load deeper detail on demand.

Prefer:

- counts, summaries, and status signals before full detail
- recent or relevant subsets before unbounded lists
- first-screen navigation that narrows scope before broad queries
- explicit "view details", "load more", or secondary routes for expensive data

Avoid treating every first screen as if it must answer every possible follow-up question immediately.

## Coordinate UX And Cost

UI and performance should be reviewed together when changing:

- default routes
- first-load tabs
- page composition
- dashboard contents
- filters and search behavior
- what loads eagerly versus after user intent

A small UX change can meaningfully reduce:

- database query count
- join and preload volume
- lock contention or connection pool pressure
- repeated computation for users who never needed the heavy state

## Avoid

Avoid choices like:

- redirecting all users to the heaviest page because it is the most impressive screen
- loading analytics, exports, and operational data together on first paint
- making a detailed index page the only route to a simple next action
- fetching broad related data before the user has selected the specific record or slice they care about

These patterns turn optional cost into mandatory cost.

## Exceptions

- Use the heavier default when it is clearly the main task and the lighter page would add friction or confusion.
- Keep critical information on the first screen when hiding it would harm safety, trust, or task success.
- Accept higher cost when the workflow is rare, high-value, and intentionally detail-heavy.
- Do not add unnecessary clicks just to save resources when the user genuinely needs the heavier page immediately.

## Review Questions

- Does the default page or tab trigger expensive work that many users do not need yet?
- Could a lighter overview or queue page get the user oriented first?
- Are we loading broad detail before the user has chosen a narrower path?
- Is this expensive query tied to clear user intent, or only to page load?
- Would a summary-first flow preserve the user outcome with lower system cost?
- Are we treating optional detail as mandatory first-load work?
