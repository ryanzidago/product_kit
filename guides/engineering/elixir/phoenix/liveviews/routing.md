# Routing

## Rule

For new LiveView CRUD-style pages, follow the newer Phoenix generator pattern:

- `SomeLive.Index` for `:index`
- `SomeLive.Form` for `:new`
- `SomeLive.Show` for `:show`
- `SomeLive.Form` for `:edit`

Keep the route action aligned with the page meaning:

- `:index` for list pages
- `:new` for create pages
- `:show` for detail pages
- `:edit` for edit pages

## Default Shape

```elixir
live "/things", ThingLive.Index, :index
live "/things/new", ThingLive.Form, :new
live "/things/:id", ThingLive.Show, :show
live "/things/:id/edit", ThingLive.Form, :edit
```

This is the default to reach for unless the page is clearly not CRUD-shaped.

## Why

- `Index` stays focused on collection pages
- `Form` keeps create and edit form concerns together when the UI is substantially the same
- `Show` keeps detail-page concerns separate
- routes read predictably across features

## Use Separate Modules Intentionally

Split away from `Form` only when the create and edit experiences are meaningfully different.

Reasonable examples:

- `SomeLive.New` when create is a wizard, chooser, or onboarding flow
- `SomeLive.Edit` when edit has different permissions, layout, or side effects
- nested modules such as `ForecastingLive.RunNew` when the route is for a distinct sub-resource

Do not create `New` and `Edit` modules just because both actions exist.

## Avoid

- putting `:new` and `:edit` back onto `Index` for new work
- mixing unrelated names like `New` for create and `EditForm` for edit in the same feature
- using `Show` for edit routes unless the edit UI is genuinely a show-page mode

## Review Questions

- Does the route follow `Index / Form / Show / Form` by default?
- If `New` or `Edit` is split out, is there a real behavioural reason?
- Will another developer be able to predict the route module name before opening the router?
